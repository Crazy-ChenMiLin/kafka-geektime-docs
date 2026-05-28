# Kafka 专栏爬虫经验总结

## 概述

本文档记录了爬取极客时间《Kafka 核心技术与实战》专栏的完整过程和经验教训。

## 目标网站

```
https://uaxe.github.io/geektime-docs/%E5%90%8E%E7%AB%AF-%E6%9E%B6%E6%9E%84/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/
```

## 遇到的问题

### 问题一：URL 编码错误

**现象**：手动构造的 URL 大量返回 404 Not Found

**原因**：
- 中文字符需要正确的 URL 编码
- 章节标题中的特殊字符（如空格、问号）需要转义

**示例**：
```
错误：https://example.com/01 消息引擎系统ABC
正确：https://example.com/01%20-%20%20%E6%B6%88%E6%81%AF%E5%BC%95%E6%93%8E%E7%B3%BB%E7%BB%9FABC
```

### 问题二：章节编号不连续

**现象**：从目录看应该有 42 章，但实际只爬取到部分章节

**原因**：
- 网站导航只展示了部分章节链接
- 章节编号跳跃：01-06, 18, 23, 26, 29, 32, 33, 36-42

**解决方案**：从首页 HTML 中提取真实链接，而非假设连续编号

### 问题三：HTML 内容提取复杂

**现象**：下载的是完整 HTML 页面，包含大量无关内容

**原因**：目标网站是 MkDocs 生成的静态站点

**解决方案**：提取 `<article>` 或 `<main>` 标签内的内容

## 解决方案

### 步骤一：抓取首页获取真实链接

```powershell
# 下载首页 HTML
$response = Invoke-WebRequest -Uri $baseUrl -UseBasicParsing
$content = $response.Content

# 使用正则提取所有章节链接
$pattern = 'href="\.\./.*?Kafka[^"]+"'
$links = [regex]::Matches($content, $pattern) | ForEach-Object { $_.Value }
```

### 步骤二：批量下载章节

```powershell
$baseUrl = "https://uaxe.github.io/geektime-docs/%E5%90%8E%E7%AB%AF-%E6%9E%B6%E6%9E%84/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98"

foreach ($link in $validLinks) {
    $fullUrl = $baseUrl + $link.Replace('../', '/')
    $response = Invoke-WebRequest -Uri $fullUrl -UseBasicParsing
    $title = [regex]::Match($response.Content, '<title>(.*?)</title>').Groups[1].Value
    $fileName = "{0:D3}-{1}.md" -f $i, ($title -replace '[\\/:*?"<>|]', '_')
    Set-Content -Path $fileName -Value $response.Content -Encoding UTF8
    $i++
}
```

### 步骤三：数据清洗

```powershell
# 删除重复文件
Get-ChildItem -Filter "*重复*" | Remove-Item

# 按序号重命名
$files = Get-ChildItem -Filter "*.md" | Sort-Object Name
$i = 1
foreach ($file in $files) {
    $newName = "{0:D3}-{1}" -f $i, $file.Name
    Rename-Item -Path $file.FullName -NewName $newName
    $i++
}
```

## 技术要点

### URL 处理
- 使用 `[Uri]::EscapeUriString()` 处理中文字符
- 注意相对路径 `../` 的转换
- 处理特殊字符：空格 → `%20`，问号 → `%3F`

### 正则表达式
- 提取链接：`href="\.\./[^"]+"`
- 提取标题：`<title>(.*?)</title>`
- 清理非法字符：`[\\/:*?"<>|]`

### 错误处理
- 使用 try-catch 包装下载请求
- 跳过 404 页面
- 记录失败的 URL 便于后续处理

## 最终成果

成功爬取 **21 个章节**，文件结构：

```
Kafka/
├── 001-Kafka核心技术与实战.md
├── 002-开篇词 为什么要学习Kafka？.md
├── 003-01 消息引擎系统ABC.md
├── ...
└── 021-期末测试.md
```

## 注意事项

1. **遵守网站规则**：设置合理的请求间隔，避免对服务器造成压力
2. **版权合规**：爬虫内容仅供个人学习使用
3. **数据完整性**：定期检查是否有新增内容
4. **异常处理**：网络波动时自动重试

## 工具推荐

| 工具 | 用途 |
|------|------|
| PowerShell | 脚本编写、批量操作 |
| Invoke-WebRequest | HTTP 请求 |
| GitHub MCP | 代码托管 |

## License

MIT License
