# 集成swag

安装依赖

```shell
go install github.com/swaggo/swag/cmd/swag@v1.8.12

go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files

// 生成api文档
swag init

浏览器访问
http://ip:host/swagger/index.html
```

**修改主启动类在主启动类上方添加如下代码**

```go
import _ "gvb_server/docs"
// @title API文档
// @version 1.0
// @description 肖晓恋爱星球API文档
// @host 127.0.0.1:8080
// @BasePath /
func main(){}
```

**修改路由引入swag**

```go
import (
	swaggerFiles "github.com/swaggo/files"
	gs "github.com/swaggo/gin-swagger"
)
router.GET("/swagger/*any", gs.WrapHandler(swaggerFiles.Handler))
```

## 广告管理的增删改查API文档

```go
// AdvertCreateView 创建广告
// @Tags 广告管理API
// @Summary 创建广告
// @Description 创建广告
// @Param data body AdvertRequest    true  "表示多个参数"
// @Param token header string  true  "token"
// @Router /api/advert [post]
// @Produce json
// @Success 200 {object} res.Response{}

// AdvertRemoveView 批量删除广告
// @Tags 广告管理API
// @Summary 批量删除广告
// @Description 批量删除广告
// @Param token header string  true  "token"
// @Param data body models.RemoveRequest    true  "广告id列表"
// @Router /api/advert [delete]
// @Produce json
// @Success 200 {object} res.Response{}

// AdvertUpdateView 修改广告
// @Tags 广告管理API
// @Summary 更新广告
// @Param token header string  true  "token"
// @Description 更新广告
// @Param data body AdvertRequest    true  "广告的一些参数"
// @Param id path int true "id"
// @Router /api/advert/{id} [put]
// @Produce json
// @Success 200 {object} res.Response{}

// AdvertListView 查询广告列表
// @Tags 广告管理API
// @Summary 广告列表
// @Description 广告列表
// @Param data query models.PageInfo    false  "查询参数"
// @Router /api/advert [get]
// @Produce json
// @Success 200 {object} res.Response{data=res.ListResponse[models.AdvertModel]}
```

> 更多方法访问`https://github.com/swaggo/swag/blob/master/README_zh-CN.md`