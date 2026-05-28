# 邀请系统业务规则

## 规则概览

| 规则项 | 说明 | 配置值 |
|-------|------|--------|
| 邀请码有效期 | 生成后有效时间 | 7 天 |
| 单用户最大邀请数 | 老用户可邀请人数上限 | 10 人 |
| 邀请奖励发放时机 | 新用户完成首单后 | 订单完成后 30 分钟 |
| 邀请者奖励 | 现金红包 | 10 元 |
| 被邀请者奖励 | 优惠券 | 15 元无门槛券 |
| 奖励有效期 | 发放后可用时长 | 7 天 |

## 业务流程

### 1. 生成邀请码

**触发条件**：
- 用户主动点击"邀请好友"按钮
- 用户注册时间 >= 7 天
- 用户完成至少 1 笔订单

**生成规则**：
```
邀请码格式：8 位字符（数字 + 大写字母）
示例：A3F8K9M2

生成算法：
1. 使用用户 ID + 时间戳 + 随机数生成种子
2. Base32 编码
3. 取前 8 位
4. 检查唯一性，冲突则重新生成
```

**实现代码**：
```kotlin
fun generateInviteCode(userId: Long): String {
    val seed = "$userId-${System.currentTimeMillis()}-${Random.nextInt()}"
    val hash = MessageDigest.getInstance("SHA-256").digest(seed.toByteArray())
    val encoded = Base32().encodeToString(hash)

    var code = encoded.substring(0, 8).uppercase()
    var attempts = 0

    while (inviteCodeExists(code) && attempts < 10) {
        code = encoded.substring(attempts, attempts + 8).uppercase()
        attempts++
    }

    if (inviteCodeExists(code)) {
        throw BusinessException("Failed to generate unique code")
    }

    // 存储到 Redis，7 天过期
    redisTemplate.opsForValue().set(
        "invite:code:$code",
        userId.toString(),
        7,
        TimeUnit.DAYS
    )

    return code
}
```

### 2. 绑定邀请关系

**触发条件**：
- 新用户注册时输入邀请码
- 邀请码未过期
- 新用户未被其他人邀请

**校验规则**：
```kotlin
fun validateInviteBinding(inviteeId: Long, inviteCode: String): ValidationResult {
    // 1. 邀请码是否存在
    val inviterId = redisTemplate.opsForValue().get("invite:code:$inviteCode")?.toLong()
        ?: return ValidationResult.fail("邀请码不存在或已过期")

    // 2. 不能邀请自己
    if (inviterId == inviteeId) {
        return ValidationResult.fail("不能使用自己的邀请码")
    }

    // 3. 是否已被邀请
    val existingInviter = getExistingInviter(inviteeId)
    if (existingInviter != null) {
        return ValidationResult.fail("您已被用户 $existingInviter 邀请")
    }

    // 4. 邀请者邀请人数是否达上限
    val inviteCount = countInvitees(inviterId)
    if (inviteCount >= MAX_INVITES) {
        return ValidationResult.fail("该邀请码已达使用上限")
    }

    // 5. 检查邀请者账户状态
    val inviter = userService.getUser(inviterId)
    if (inviter.status != UserStatus.ACTIVE) {
        return ValidationResult.fail("邀请码无效")
    }

    return ValidationResult.success()
}
```

**绑定流程**：
```kotlin
@Transactional
fun bindInviteRelation(inviteeId: Long, inviteCode: String) {
    // 1. 校验
    val validation = validateInviteBinding(inviteeId, inviteCode)
    if (!validation.isSuccess) {
        throw BusinessException(validation.message)
    }

    val inviterId = getInviterFromCode(inviteCode)

    // 2. 保存邀请关系（数据库）
    val relation = InviteRelation(
        inviterId = inviterId,
        inviteeId = inviteeId,
        inviteCode = inviteCode,
        status = InviteStatus.PENDING,
        createTime = LocalDateTime.now()
    )
    inviteRelationRepository.save(relation)

    // 3. 缓存邀请关系（Redis，永久）
    redisTemplate.opsForValue().set(
        "user:inviter:$inviteeId",
        inviterId.toString()
    )

    // 4. 增加邀请者的邀请计数
    redisTemplate.opsForValue().increment("user:invite:count:$inviterId")

    // 5. 发放新用户注册奖励（优惠券）
    couponService.issueCoupon(
        userId = inviteeId,
        couponTemplateId = NEW_USER_COUPON_TEMPLATE,
        source = "INVITE_REGISTER"
    )

    // 6. 发送通知
    notificationService.send(
        userId = inviterId,
        type = NotificationType.INVITE_SUCCESS,
        content = "您的好友已注册成功，完成首单后即可获得奖励"
    )
}
```

### 3. 首单触发奖励

**触发时机**：
- 新用户订单状态变更为"已完成"
- 订单为用户首单
- 订单金额 >= 1 元

**发放流程**：
```kotlin
@Async
fun onFirstOrderComplete(orderId: Long, userId: Long) {
    // 1. 验证是否为首单
    val orderCount = orderRepository.countByUserId(userId)
    if (orderCount > 1) {
        logger.info("Not first order, skip reward: user=$userId")
        return
    }

    // 2. 查询邀请关系
    val inviterId = getInviter(userId) ?: run {
        logger.info("No inviter found for user: $userId")
        return
    }

    // 3. 防重复发放检查
    val rewardKey = "invite:reward:$inviterId:$userId"
    val locked = redisTemplate.opsForValue().setIfAbsent(
        rewardKey,
        "1",
        30,
        TimeUnit.DAYS
    ) ?: false

    if (!locked) {
        logger.warn("Reward already granted: inviter=$inviterId, invitee=$userId")
        return
    }

    try {
        // 4. 发放邀请者奖励（现金红包）
        walletService.deposit(
            userId = inviterId,
            amount = INVITER_REWARD_AMOUNT,
            bizType = BizType.INVITE_REWARD,
            bizId = userId.toString(),
            remark = "邀请好友完成首单奖励"
        )

        // 5. 更新邀请关系状态
        inviteRelationRepository.updateStatus(
            inviterId = inviterId,
            inviteeId = userId,
            status = InviteStatus.REWARDED,
            firstOrderTime = LocalDateTime.now()
        )

        // 6. 发送通知
        notificationService.send(
            userId = inviterId,
            type = NotificationType.REWARD_GRANTED,
            content = "您邀请的好友已完成首单，奖励已发放"
        )

        // 7. 埋点统计
        analyticsService.track(
            event = "invite_reward_granted",
            properties = mapOf(
                "inviter_id" to inviterId,
                "invitee_id" to userId,
                "order_id" to orderId,
                "reward_amount" to INVITER_REWARD_AMOUNT
            )
        )

    } catch (e: Exception) {
        logger.error("Failed to grant invite reward", e)
        // 释放锁，允许重试
        redisTemplate.delete(rewardKey)
        throw e
    }
}
```

## 特殊场景处理

### 1. 作弊行为识别

**识别规则**：
```kotlin
fun detectFraud(inviterId: Long, inviteeId: Long): FraudCheckResult {
    val checks = mutableListOf<String>()

    // 1. 设备指纹相同
    val inviterDevice = getDeviceId(inviterId)
    val inviteeDevice = getDeviceId(inviteeId)
    if (inviterDevice == inviteeDevice) {
        checks.add("SAME_DEVICE")
    }

    // 2. 注册 IP 相同且短时间内
    val inviterIp = getUserRegisterIp(inviterId)
    val inviteeIp = getUserRegisterIp(inviteeId)
    val timeDiff = getRegisterTimeDiff(inviterId, inviteeId)
    if (inviterIp == inviteeIp && timeDiff < Duration.ofMinutes(10)) {
        checks.add("SAME_IP_SHORT_TIME")
    }

    // 3. 手机号相似（前 7 位相同）
    val inviterPhone = getUserPhone(inviterId)
    val inviteePhone = getUserPhone(inviteeId)
    if (inviterPhone.substring(0, 7) == inviteePhone.substring(0, 7)) {
        checks.add("SIMILAR_PHONE")
    }

    // 4. 邀请者短时间内邀请多人
    val recentInvites = getRecentInvites(inviterId, Duration.ofHours(1))
    if (recentInvites.size > 3) {
        checks.add("HIGH_FREQUENCY_INVITE")
    }

    // 5. 新用户首单金额异常低（可能刷单）
    val firstOrderAmount = getFirstOrderAmount(inviteeId)
    if (firstOrderAmount != null && firstOrderAmount < BigDecimal("5.00")) {
        checks.add("LOW_FIRST_ORDER_AMOUNT")
    }

    return FraudCheckResult(
        isFraud = checks.isNotEmpty(),
        reasons = checks,
        riskLevel = calculateRiskLevel(checks.size)
    )
}
```

**处理策略**：
- **低风险**（1 个指标异常）：正常发放，记录日志
- **中风险**（2 个指标异常）：延迟 24 小时发放，人工审核
- **高风险**（3+ 个指标异常）：拒绝发放，冻结邀请功能

```kotlin
fun handleFraudResult(result: FraudCheckResult, inviterId: Long, inviteeId: Long) {
    when (result.riskLevel) {
        RiskLevel.LOW -> {
            logger.info("Low risk detected: $result")
            // 正常发放
        }
        RiskLevel.MEDIUM -> {
            logger.warn("Medium risk detected: $result")
            // 延迟发放
            delayRewardGrant(inviterId, inviteeId, Duration.ofHours(24))
            // 创建审核工单
            createReviewTicket(inviterId, inviteeId, result)
        }
        RiskLevel.HIGH -> {
            logger.error("High risk detected: $result")
            // 拒绝发放
            rejectReward(inviterId, inviteeId, result)
            // 冻结邀请功能
            freezeInviteFeature(inviterId, Duration.ofDays(7))
            // 通知风控团队
            notifyRiskTeam(inviterId, inviteeId, result)
        }
    }
}
```

### 2. 补发奖励

**适用场景**：
- 系统故障导致奖励未发放
- 用户申诉后确认应发放

**操作流程**：
```kotlin
@Transactional
fun manualGrantReward(inviterId: Long, inviteeId: Long, operator: String, reason: String) {
    // 1. 验证邀请关系
    val relation = inviteRelationRepository.findByInviterAndInvitee(inviterId, inviteeId)
        ?: throw BusinessException("邀请关系不存在")

    // 2. 检查是否已发放
    val existingReward = rewardRecordRepository.findByInviterAndInvitee(inviterId, inviteeId)
    if (existingReward != null) {
        throw BusinessException("奖励已发放，记录 ID: ${existingReward.id}")
    }

    // 3. 发放奖励
    walletService.deposit(
        userId = inviterId,
        amount = INVITER_REWARD_AMOUNT,
        bizType = BizType.INVITE_REWARD_MANUAL,
        bizId = inviteeId.toString(),
        remark = "人工补发邀请奖励: $reason"
    )

    // 4. 记录操作日志
    auditLogService.log(
        operator = operator,
        action = "MANUAL_GRANT_REWARD",
        target = "invite_reward",
        details = mapOf(
            "inviter_id" to inviterId,
            "invitee_id" to inviteeId,
            "amount" to INVITER_REWARD_AMOUNT,
            "reason" to reason
        )
    )

    // 5. 更新邀请关系状态
    relation.status = InviteStatus.REWARDED
    relation.updateTime = LocalDateTime.now()
    inviteRelationRepository.save(relation)
}
```

### 3. 奖励撤回

**适用场景**：
- 发现作弊行为，撤回已发放奖励
- 新用户退款，触发奖励回收

**操作流程**：
```kotlin
@Transactional
fun revokeReward(inviterId: Long, inviteeId: Long, reason: String) {
    // 1. 查询奖励记录
    val reward = rewardRecordRepository.findByInviterAndInvitee(inviterId, inviteeId)
        ?: throw BusinessException("奖励记录不存在")

    if (reward.status == RewardStatus.REVOKED) {
        throw BusinessException("奖励已撤回")
    }

    // 2. 扣减账户余额
    try {
        walletService.withdraw(
            userId = inviterId,
            amount = reward.amount,
            bizType = BizType.INVITE_REWARD_REVOKE,
            bizId = inviteeId.toString(),
            remark = "撤回邀请奖励: $reason"
        )
    } catch (e: InsufficientBalanceException) {
        // 余额不足，记录欠款
        debtService.recordDebt(
            userId = inviterId,
            amount = reward.amount,
            reason = reason
        )
    }

    // 3. 更新奖励记录状态
    reward.status = RewardStatus.REVOKED
    reward.revokeTime = LocalDateTime.now()
    reward.revokeReason = reason
    rewardRecordRepository.save(reward)

    // 4. 发送通知
    notificationService.send(
        userId = inviterId,
        type = NotificationType.REWARD_REVOKED,
        content = "您的邀请奖励因「$reason」被撤回"
    )
}
```

## 数据统计

### 邀请漏斗分析

```sql
-- 邀请漏斗
SELECT
    '生成邀请码' AS stage,
    COUNT(DISTINCT user_id) AS user_count
FROM invite_codes
WHERE create_time >= '2024-05-01'

UNION ALL

SELECT
    '邀请码被使用' AS stage,
    COUNT(DISTINCT inviter_id) AS user_count
FROM invite_relations
WHERE create_time >= '2024-05-01'

UNION ALL

SELECT
    '新用户完成注册' AS stage,
    COUNT(DISTINCT invitee_id) AS user_count
FROM invite_relations
WHERE create_time >= '2024-05-01'

UNION ALL

SELECT
    '新用户完成首单' AS stage,
    COUNT(DISTINCT invitee_id) AS user_count
FROM invite_relations
WHERE first_order_time >= '2024-05-01'

UNION ALL

SELECT
    '奖励发放' AS stage,
    COUNT(DISTINCT inviter_id) AS user_count
FROM reward_records
WHERE grant_time >= '2024-05-01'
AND reward_type = 'INVITE_REWARD';
```

### 邀请效果分析

```sql
-- Top 邀请者
SELECT
    inviter_id,
    COUNT(*) AS invite_count,
    SUM(CASE WHEN first_order_time IS NOT NULL THEN 1 ELSE 0 END) AS success_count,
    SUM(CASE WHEN first_order_time IS NOT NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS success_rate
FROM invite_relations
WHERE create_time >= '2024-05-01'
GROUP BY inviter_id
ORDER BY success_count DESC
LIMIT 100;

-- 邀请用户留存率
SELECT
    DATE(r.create_time) AS invite_date,
    COUNT(DISTINCT r.invitee_id) AS total_invitees,
    COUNT(DISTINCT o.user_id) AS day1_active,
    COUNT(DISTINCT o7.user_id) AS day7_active,
    COUNT(DISTINCT o30.user_id) AS day30_active
FROM invite_relations r
LEFT JOIN orders o ON o.user_id = r.invitee_id
    AND DATE(o.create_time) = DATE(r.create_time) + INTERVAL 1 DAY
LEFT JOIN orders o7 ON o7.user_id = r.invitee_id
    AND DATE(o7.create_time) = DATE(r.create_time) + INTERVAL 7 DAY
LEFT JOIN orders o30 ON o30.user_id = r.invitee_id
    AND DATE(o30.create_time) = DATE(r.create_time) + INTERVAL 30 DAY
WHERE r.create_time >= '2024-05-01'
GROUP BY DATE(r.create_time)
ORDER BY invite_date DESC;
```

## FAQ

**Q1: 邀请码可以重复使用吗？**
A: 可以。单个邀请码可以被多人使用，直到邀请者达到邀请人数上限（10 人）。

**Q2: 新用户注册后多久必须完成首单？**
A: 没有时间限制。但邀请码有 7 天有效期，超期后新用户注册时无法使用。

**Q3: 如果新用户退款，奖励会被回收吗？**
A: 会。新用户首单退款后，系统会自动撤回邀请奖励。

**Q4: 邀请奖励何时到账？**
A: 新用户首单完成后 30 分钟内到账。如未到账，请联系客服。

**Q5: 被邀请者的优惠券何时发放？**
A: 注册成功后立即发放，有效期 7 天。

## 相关文档

- [营销活动平台架构](../SKILL.md)
- [风控系统对接](../../risk-control/SKILL.md)
- [钱包服务 API](../../payment-system/docs/wallet-api.md)
