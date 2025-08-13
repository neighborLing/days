# Bug列表获取自动化指南 🐛

## 📋 概述
本文档记录了通过自动化方式获取蓝凌系统bug列表的完整流程，包括无头浏览器配置、登录信息管理和数据提取逻辑。

## 🔧 技术方案

### 1. 无头浏览器配置
```javascript
// Puppeteer无头浏览器启动配置
const launchOptions = {
  headless: true,  // 无头模式，提高性能
  args: [
    '--no-sandbox',
    '--disable-setuid-sandbox',
    '--disable-dev-shm-usage',
    '--disable-accelerated-2d-canvas',
    '--no-first-run',
    '--no-zygote',
    '--disable-gpu'
  ]
};
```

### 2. 登录信息管理

#### 登录凭据存储
```json
{
  "蓝凌系统登录信息": {
    "url": "http://kl.landray.com.cn/#/agile/work-list?activeKey=issue&category=AGILE&id=283986611375841280&name=MK%E7%A0%94%E5%8F%91%E9%A1%B9%E7%9B%AE%E7%AE%A1%E7%90%86&organizationId=1&type=project",
    "username": "lingt1",
    "password": "WYXlalala1.",
    "selectors": {
      "usernameField": "#username",
      "passwordField": "#password",
      "loginButton": "button.c7n-btn.c7n-btn-primary"
    },
    "lastLoginTime": "2025-08-13",
    "sessionValid": false
  }
}
```

#### 会话管理策略
1. **首次登录**: 使用用户名密码登录
2. **会话检查**: 每次访问前检查是否已登录
3. **自动重登**: 会话失效时自动重新登录
4. **Cookie保存**: 保存登录状态以减少重复登录

## 🚀 自动化流程

### 3. Bug列表获取逻辑

#### 步骤1: 初始化浏览器
```javascript
// 启动无头浏览器
const browser = await puppeteer.launch(launchOptions);
const page = await browser.newPage();
```

#### 步骤2: 登录检查与处理
```javascript
// 导航到目标页面
await page.goto(targetUrl);

// 检查是否需要登录
const needLogin = await page.$('#username') !== null;

if (needLogin) {
  // 执行登录流程
  await page.type('#username', credentials.username);
  await page.type('#password', credentials.password);
  await page.click('button.c7n-btn.c7n-btn-primary');
  
  // 等待登录完成
  await page.waitForNavigation();
}
```

#### 步骤3: 切换到列表视图
```javascript
// 点击树形视图切换
await page.click('input.c7n-pro-select[value="树形视图"]');

// 选择列表视图
await page.click('li.c7n-pro-select-dropdown-menu-item[role="menuitem"][aria-selected="false"]');

// 等待页面加载
await page.waitForTimeout(2000);
```

#### 步骤4: 数据提取
```javascript
const bugData = await page.evaluate(() => {
  const today = new Date().toISOString().split('T')[0];
  const rows = document.querySelectorAll('[role="row"]:not([role="row"]:first-child)');
  
  const bugs = [];
  
  rows.forEach((row, index) => {
    const rowText = row.textContent || '';
    
    // 提取bug编号
    const idMatch = rowText.match(/MKR-\d+/);
    const bugId = idMatch ? idMatch[0] : '';
    
    if (bugId) {
      // 提取其他信息
      const dateMatches = rowText.match(/\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}/g);
      const priorityMatch = rowText.match(/(紧急|高|中|低)/);
      const statusMatch = rowText.match(/(待开发|待处理|待入库|已完成|进行中)/);
      const summaryMatch = rowText.match(/^(.+?)MKR-\d+/);
      
      const bugInfo = {
        id: bugId,
        summary: summaryMatch ? summaryMatch[1].trim() : '',
        priority: priorityMatch ? priorityMatch[0] : '',
        status: statusMatch ? statusMatch[0] : '',
        updateTime: dateMatches && dateMatches[0] ? dateMatches[0] : '',
        createTime: dateMatches && dateMatches[1] ? dateMatches[1] : '',
        assignee: rowText.includes('凌通') ? '凌通' : ''
      };
      
      bugs.push(bugInfo);
    }
  });
  
  return {
    totalCount: bugs.length,
    bugs: bugs,
    extractTime: new Date().toISOString()
  };
});
```

## 📊 数据处理与分析

### 4. 今日Bug筛选
```javascript
function filterTodayBugs(allBugs) {
  const today = new Date().toISOString().split('T')[0];
  
  return allBugs.filter(bug => {
    const createDate = bug.createTime.split(' ')[0];
    const updateDate = bug.updateTime.split(' ')[0];
    return createDate === today || updateDate === today;
  });
}
```

### 5. 统计分析
```javascript
function generateBugSummary(bugs) {
  return {
    total: bugs.length,
    priorityDistribution: {
      urgent: bugs.filter(b => b.priority === '紧急').length,
      high: bugs.filter(b => b.priority === '高').length,
      medium: bugs.filter(b => b.priority === '中').length,
      low: bugs.filter(b => b.priority === '低').length
    },
    statusDistribution: {
      pending: bugs.filter(b => b.status === '待开发').length,
      processing: bugs.filter(b => b.status === '待处理').length,
      toStore: bugs.filter(b => b.status === '待入库').length
    }
  };
}
```

## 🔄 优化建议

### 6. 性能优化
- 使用无头浏览器减少资源消耗
- 实现会话复用避免重复登录
- 添加请求缓存机制
- 设置合理的等待时间

### 7. 错误处理
```javascript
try {
  // bug获取逻辑
} catch (error) {
  console.error('Bug列表获取失败:', error);
  
  // 重试机制
  if (retryCount < maxRetries) {
    await delay(retryDelay);
    return getBugList(retryCount + 1);
  }
  
  throw error;
}
```

### 8. 定时任务配置
```javascript
// 每日定时获取bug列表
const schedule = require('node-schedule');

// 每天上午9点执行
schedule.scheduleJob('0 9 * * *', async () => {
  try {
    const bugData = await getBugList();
    await saveBugReport(bugData);
    console.log('每日bug报告生成完成');
  } catch (error) {
    console.error('定时任务执行失败:', error);
  }
});
```

## 📝 使用示例

### 9. 完整调用示例
```javascript
async function dailyBugReport() {
  const browser = await puppeteer.launch({ headless: true });
  
  try {
    const bugData = await getBugList(browser);
    const todayBugs = filterTodayBugs(bugData.bugs);
    const summary = generateBugSummary(bugData.bugs);
    
    console.log(`今日Bug活动: ${todayBugs.length}个`);
    console.log(`总Bug数量: ${summary.total}个`);
    console.log('优先级分布:', summary.priorityDistribution);
    
    return {
      todayBugs,
      summary,
      timestamp: new Date().toISOString()
    };
  } finally {
    await browser.close();
  }
}
```

## 🔐 安全注意事项

### 10. 凭据安全
- 使用环境变量存储敏感信息
- 定期更新密码
- 实现访问日志记录
- 避免在代码中硬编码凭据

---

**最后更新时间**: 2025-08-13  
**维护人员**: 凌通  
**版本**: v1.0