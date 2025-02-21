# review
一个电影点评平台的后端系统。该项目旨在为用户提供一个便捷的电影点评和互动平台，支持用户登录、查找店铺、秒杀优惠券、发表点评、关注好友，共同关注等功能

1.前后端分离，单体项目

2.后端部署在tomcat服务器上，前端部署在nginx上，前端向nginx请求页面（静态资源），页面通过Ajax请求向tomcat服务器请求数据（mysql,redis），返回给前端

3.考虑到后续水平扩展问题，tomcat可以集群部署，要考虑到并发，数据共享等等问题

**注意：**

使用redis存储数据一定要注意设置过期时间，数据数据结构，key的唯一性，类型，存储粒度（不用全部信息）

