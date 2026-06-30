# 天气可视化仪表盘 Axios + ECharts5.x
## 项目简介
本项目基于原生JS、Axios、ECharts5、Open-Meteo免费气象API实现多图表联动天气仪表盘，完成链式异步请求、响应式布局、全局Loading、异常捕获等全部任务要求。

## 技术栈
- Axios：处理HTTP链式异步请求，全局拦截统一管理Loading状态
- ECharts 5.x：折线图、柱状图、雷达风向玫瑰图，多图表connect联动
- Open-Meteo API：免费地理编码+7天逐时气象数据，无需密钥
- 原生CSS Grid：响应式布局，适配手机/平板/桌面端

## 核心功能（全部满足任务硬性指标）
1. Axios链式异步流程
   - async/await 串行调用【城市→经纬度】→【经纬度→气象数据】接口
   - axios拦截器统一显示/隐藏Loading遮罩
   - try/catch捕获网络错误、城市不存在等异常，友好提示
2. ECharts多图表联动
   - 温度折线图、降雨量柱状图、风向玫瑰雷达图三种图表
   - echarts.connect()实现多图共享dataZoom，hover同步高亮
3. 响应式仪表盘
   - CSS Grid自动分栏，移动端单列、桌面双列
   - window.resize监听自动重绘图表
4. 完整交互
   - 输入城市回车/按钮查询、切换城市实时刷新图表
   - 网络/参数错误统一提示框


## 运行方式
1. 将index.html直接使用浏览器打开（无需后端，纯前端项目）
2. 默认加载北京7天天气，输入任意国内城市即可切换数据
3. 缩放浏览器窗口，图表自动适配尺寸

## 文件说明
- index.html：整合HTML结构、CSS样式、全部JS业务逻辑
- README.md：项目说明文档（本文件）

[2.html](https://github.com/user-attachments/files/29489226/2.html)

