日期：2025-03-05
## 带入群验证功能的NoneBot2插件
```python
import random
from datetime import datetime, timedelta
from nonebot import on_request, on_message, on_notice
from nonebot.adapters.onebot.v11 import Bot, GroupRequestEvent, GroupMessageEvent, GroupDecreaseNoticeEvent
from nonebot.typing import T_State
from nonebot.permission import SUPERUSER

# 存储验证码信息，键为用户 ID，值为 (验证码, 过期时间, 群 ID)
verification_codes = {}
# 存储被拉黑的用户 ID
blacklisted_users = set()

# 自动同意加群请求
group_request = on_request()


@group_request.handle()
async def handle_group_request(bot: Bot, event: GroupRequestEvent):
    user_id = event.user_id
    if user_id in blacklisted_users:
        await bot.set_group_add_request(
            flag=event.flag,
            sub_type=event.sub_type,
            approve=False,
            reason="你已被拉黑，无法加入本群"
        )
        return

    if event.request_type == 'group' and event.sub_type == 'add':
        await bot.set_group_add_request(
            flag=event.flag,
            sub_type=event.sub_type,
            approve=True
        )
        # 生成 6 位数字验证码
        code = str(random.randint(100000, 999999))
        # 设置过期时间为 3 分钟（180 秒）后
        expiration_time = datetime.now() + timedelta(seconds=180)
        verification_codes[user_id] = (code, expiration_time, event.group_id)
        # 发送带 @ 的验证码消息
        at_user = f"[CQ:at,qq={user_id}]"
        await bot.send_group_msg(
            group_id=event.group_id,
            message=f"{at_user} 欢迎加入本群，请在 3 分钟内回复验证码：{code}"
        )


# 验证用户输入的验证码
group_message = on_message()


@group_message.handle()
async def handle_group_message_verification(bot: Bot, event: GroupMessageEvent, state: T_State):
    user_id = event.user_id
    message = event.get_plaintext()

    if user_id in verification_codes:
        code, expiration_time, group_id = verification_codes[user_id]
        if datetime.now() > expiration_time:
            # 验证码过期，踢出用户
            del verification_codes[user_id]
            at_user = f"[CQ:at,qq={user_id}]"
            await bot.set_group_kick(
                group_id=group_id,
                user_id=user_id,
                reject_add_request=False
            )
            await bot.send_group_msg(
                group_id=group_id,
                message=f"{at_user} 验证码超时，已踢出群聊"
            )
        elif message == code:
            # 验证码验证成功
            del verification_codes[user_id]
            at_user = f"[CQ:at,qq={user_id}]"
            await bot.send_group_msg(
                group_id=group_id,
                message=f"{at_user} 验证码验证成功，欢迎你成为本群的一员！禁止讨论一切金额卡相关的话题！"
            )
        else:
            # 验证码验证失败，踢出用户
            del verification_codes[user_id]
            at_user = f"[CQ:at,qq={user_id}]"
            await bot.set_group_kick(
                group_id=group_id,
                user_id=user_id,
                reject_add_request=False
            )
            await bot.send_group_msg(
                group_id=group_id,
                message=f"{at_user} 验证码验证失败，已踢出群聊"
            )


# 定期检查未验证的用户，处理未输入验证码的情况
async def check_unverified_users(bot: Bot):
    now = datetime.now()
    for user_id, (code, expiration_time, group_id) in list(verification_codes.items()):
        if now > expiration_time:
            del verification_codes[user_id]
            at_user = f"[CQ:at,qq={user_id}]"
            await bot.set_group_kick(
                group_id=group_id,
                user_id=user_id,
                reject_add_request=False
            )
            await bot.send_group_msg(
                group_id=group_id,
                message=f"{at_user} 未在规定时间内输入验证码，已踢出群聊"
            )


# 处理群成员退群事件
group_decrease = on_notice()


@group_decrease.handle()
async def handle_group_decrease(bot: Bot, event: GroupDecreaseNoticeEvent):
    if event.sub_type == 'leave':
        user_id = event.user_id
        blacklisted_users.add(user_id)
        at_user = f"[CQ:at,qq={user_id}]"
        await bot.send_group_msg(
            group_id=event.group_id,
            message=f"{at_user} 已退群，已将其拉黑，禁止再次加入本群"
        )


# 群管理功能（踢人、禁言、解禁）
group_msg = on_message()


@group_msg.handle()
async def handle_group_management(bot: Bot, event: GroupMessageEvent):
    # 获取消息文本
    message_text = str(event.get_plaintext())
    # 获取消息中的 at 列表
    at_list = [seg.data.get('qq') for seg in event.get_message() if seg.type == 'at']

    # 踢人功能
    if "踢" in message_text and at_list:
        if event.sender.role in ['admin', 'owner'] or await SUPERUSER(bot, event):
            group_id = event.group_id
            for qq in at_list:
                if qq != 'all':
                    try:
                        await bot.set_group_kick(group_id=group_id, user_id=int(qq))
                        await group_msg.send(f"已将 {qq} 踢出群聊")
                    except Exception as e:
                        await group_msg.send(f"踢人失败：{str(e)}")
        else:
            await group_msg.send("你没有权限踢人")

    # 禁言功能
    elif "禁" in message_text and at_list:
        if event.sender.role in ['admin', 'owner'] or await SUPERUSER(bot, event):
            group_id = event.group_id
            duration = 60
            try:
                for word in message_text.split():
                    if word.isdigit():
                        duration = int(word)
                        break
            except ValueError:
                pass

            for qq in at_list:
                if qq != 'all':
                    try:
                        await bot.set_group_ban(group_id=group_id, user_id=int(qq), duration=duration)
                        await group_msg.send(f"已将 {qq} 禁言 {duration} 秒")
                    except Exception as e:
                        await group_msg.send(f"禁言失败：{str(e)}")
        else:
            await group_msg.send("你没有权限禁言")

    # 解禁功能
    elif "解" in message_text and at_list:
        if event.sender.role in ['admin', 'owner'] or await SUPERUSER(bot, event):
            group_id = event.group_id
            for qq in at_list:
                if qq != 'all':
                    try:
                        await bot.set_group_ban(group_id=group_id, user_id=int(qq), duration=0)
                        await group_msg.send(f"已将 {qq} 解禁")
                    except Exception as e:
                        await group_msg.send(f"解禁失败：{str(e)}")
        else:
            await group_msg.send("你没有权限解禁")
```
## 指令格式
@ 踢
@ 禁 空格 300 为5分钟 不输入300 则默认为1分钟
@ 解
