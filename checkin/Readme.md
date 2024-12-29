# 1. Checkin Plugin Overview

The checkin plugin implements a daily check-in system for HydroOJ users, allowing them to receive daily fortunes and track their check-in streaks.

## Imported Package Usage

```typescript
import {
    _, db, UserModel, SettingModel, DomainModel, moment, Handler, PRIV
} from 'hydrooj';
```

Let's examine how each package is used:

### 1. UserModel
Used for managing user data and check-in status:
```typescript
// Get user document
const udoc = await UserModel.getById("system", uid);

// Update user's check-in information
await UserModel.setById(uid, {
    checkin_luck: check_luck,
    checkin_time: checkin_time,
    checkin_cnt_now: checkin_cnt_now,
    checkin_cnt_all: checkin_cnt_all
});
```

### 2. moment
Handles date operations:
```typescript
const now = moment().format("YYYY-MM-DD");
// Check if consecutive days
moment(now).diff(moment(checkin_time), 'days') == 1
```

### 3. SettingModel
Manages plugin settings:
```typescript
c.setting.AccountSetting(
    SettingModel.Setting('setting_33oj', 'checkin_time', '1997-01-11', 'text'),
    SettingModel.Setting('setting_33oj', 'checkin_luck', 0, 'number'),
    SettingModel.Setting('setting_33oj', 'checkin_cnt_now', 0, 'number'),
    SettingModel.Setting('setting_33oj', 'checkin_cnt_all', 0, 'number')
);
```

# 2. Models and Handlers

## CheckinHandler
Main handler for processing check-ins:

```typescript
class CheckinHandler extends Handler {
    async get() {
        const uid = this.user._id;
        const udoc = await UserModel.getById("system", uid);
        
        // Get current check-in status
        let check_luck = udoc["checkin_luck"];
        let checkin_time = udoc["checkin_time"];
        let checkin_cnt_now = udoc["checkin_cnt_now"];
        let checkin_cnt_all = udoc["checkin_cnt_all"];
        
        const now = moment().format("YYYY-MM-DD");
        
        // Process new check-in
        if (now != checkin_time) {
            // Generate random luck value
            check_luck = Math.floor(Math.random() * 7);
            
            // Update counters
            checkin_cnt_all = checkin_cnt_all ? checkin_cnt_all + 1 : 1;
            
            // Check for consecutive days
            if (checkin_cnt_now && 
                moment(now).diff(moment(checkin_time), 'days') == 1) {
                checkin_cnt_now++;
            } else {
                checkin_cnt_now = 1;
            }
            
            // Update user data
            await UserModel.setById(uid, {
                checkin_luck,
                checkin_time: now,
                checkin_cnt_now,
                checkin_cnt_all
            });
        }
        
        this.response.redirect = "/";
    }
}
```

## Homepage Integration
```typescript
async function getCheckin(payload) {
    var today = moment().format("YYYY-MM-DD");
    payload.luck_today = today;
    return payload;
}

HomeHandler.prototype.getCheckin = async (domainId, payload) => {
    return await getCheckin(payload);
}
```

# 3. UI Modifications

The plugin modifies the homepage UI by adding a check-in section:

## Homepage Check-in Section (checkin.html)
```html
<div class="section side">
    <div class="section__header">
        <h1 class="section__title">今日运势</h1>
    </div>
    <div class="section__body typo" style="text-align: center">
        {% if UserContext._id != 0 %}
            {% if payload.luck_today == UserContext.checkin_time %}
                {% if payload.luck_vip.includes(UserContext._id) %}
                    <h1 style="font-weight: bold; color:{{payload.luck_type[0].color}};">
                        § 天天大吉 §
                    </h1>
                {% else %}
                    <h1 style="font-weight: bold; color:{{payload.luck_type[UserContext.checkin_luck].color}};">
                        § {{payload.luck_type[UserContext.checkin_luck].text}} §
                    </h1>
                {% endif %}
                <p>已连续打卡 {{UserContext.checkin_cnt_now}} 天，
                   累计打卡 {{UserContext.checkin_cnt_all}} 天</p>
            {% else %}
                <a href="/checkin" class="expanded button">
                    <span class="icon icon-feeling-lucky"></span>
                    抽取今日运势
                </a>
            {% endif %}
        {% else %}
            <p>登录后可抽取</p>
        {% endif %}
    </div>
</div>
```

## Modified Pages:

1. **Homepage**
   - Adds check-in section showing:
     - Daily fortune
     - Check-in streak
     - Total check-ins
     - Check-in button

2. **User Profile**
   - Stores check-in related data:
     - Last check-in time
     - Current luck value
     - Consecutive check-in count
     - Total check-in count

## Plugin Registration
```typescript
export async function apply(ctx: Context) {
    // Register settings
    ctx.inject(['setting'], (c) => {
        c.setting.AccountSetting(
            SettingModel.Setting('setting_33oj', 'checkin_time'),
            SettingModel.Setting('setting_33oj', 'checkin_luck'),
            SettingModel.Setting('setting_33oj', 'checkin_cnt_now'),
            SettingModel.Setting('setting_33oj', 'checkin_cnt_all')
        );
    });
    
    // Register route
    ctx.Route('checkin', '/checkin', CheckinHandler, PRIV.PRIV_USER_PROFILE);
}
```

The checkin plugin enhances user engagement by:
- Providing daily check-in rewards
- Tracking check-in streaks
- Displaying personalized fortunes
- Adding gamification elements to the platform

It integrates seamlessly with HydroOJ's existing UI while adding new interactive elements to encourage regular user participation.
