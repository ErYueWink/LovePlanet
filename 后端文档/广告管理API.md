# 广告管理API

## 新增广告

前端传来的值：

1. 广告标题
2. 点击跳转链接
3. 广告图片路径
4. 是否显示

> 其中**广告标题、点击跳转链接、广告图片路径**为必填项  
>
> 是否显示默认为`false`不显示

**步骤：**

1. 进行参数绑定
2. 根据广告标题查询广告是否重复
   - 重复：返回错误信息，提醒给用户说明广告已重复
   - 不重复：继续向下执行
3. 创建广告，广告入库

```go
package advert_api

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/models"
	"gvb_server/utils/res"
)

type AdvertRequest struct {
	Title  string `json:"title" binding:"required" msg:"请输入广告标题"  struct:"title""`    // 广告标题
	Href   string `json:"href" binding:"required" msg:"请输入广告跳转链接" struct:"href"`       // 广告跳转链接
	Images string `json:"images" binding:"required" msg:"请输入广告图片" struct:"images"`   // 广告图片
	IsShow bool   `json:"is_show" struct:"is_show"` // 是否显示
}

// AdvertCreateView 创建广告
func (AdvertApi) AdvertCreateView(c *gin.Context) {
	var cr AdvertRequest
	err := c.ShouldBindJSON(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	var advert models.AdvertModel
	count := global.DB.Take(&advert, "title = ?", cr.Title).RowsAffected
	if count != 0 {
		res.FailWithMsg(fmt.Sprintf("广告%s重复", cr.Title), c)
		return
	}
	err = global.DB.Create(&models.AdvertModel{
		Title:  cr.Title,
		Href:   cr.Href,
		Images: cr.Images,
		IsShow: cr.IsShow,
	}).Error
	if err != nil {
		res.FailWithMsg("创建广告失败", c)
		return
	}
	res.OKWithMsg("创建广告成功", c)
}

```

## 修改广告

安装结构体转`Map`的第三方包

```shell
go get github.com/fatih/structs
```

**步骤：**

1. 接收前端传来的参数
2. 根据`ID主键`查询广告是否存在
   - 存在：继续向下执行
   - 不存在：提醒用户说明要修改的广告不存在
3. 将结构体转为Map
4. 修改广告
5. 返回修改成功信息

> 修改广告时，由于我们要修改多个参数需要将结构体转为`Map`

```go
package advert_api

import (
	"github.com/fatih/structs"
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/models"
	"gvb_server/utils/res"
)

// AdvertUpdateView 修改广告
func (AdvertApi) AdvertUpdateView(c *gin.Context) {
	id := c.Param("id")
	var cr AdvertRequest
	err := c.ShouldBindJSON(&cr)
	// 参数绑定
	if err != nil {
		res.FailWithError(err, &cr, c)
		return
	}
	var advertModel models.AdvertModel
	count := global.DB.Take(&advertModel, id).RowsAffected
	if count == 0 {
		res.FailWithMsg("广告不存在", c)
		return
	}
	// 结构体转Map
	advertMap := structs.Map(&cr)
	// 修改广告
	err = global.DB.Model(&advertModel).Updates(advertMap).Error
	if err != nil {
		res.FailWithMsg("修改广告失败", c)
		return
	}
	res.OKWithMsg("修改广告成功", c)

}
```

## 删除广告

**步骤：**

1. 参数绑定
2. 根据主键列表查询判断要删除的广告是否存在
   - 存在：继续向下执行
   - 不存在：返回错误信息，提醒用户说明要删除的广告不存在
3. 批量删除广告

```go
package advert_api

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/models"
	"gvb_server/utils/res"
)

// AdvertRemoveView 删除广告
func (AdvertApi) AdvertRemoveView(c *gin.Context) {
	var cr models.RemoveRequest
	err := c.ShouldBindJSON(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	var advertList []models.AdvertModel
	count := global.DB.Find(&advertList, cr.IDList).RowsAffected
	if count == 0 {
		res.FailWithMsg("广告不存在", c)
		return
	}
	err = global.DB.Delete(&advertList).Error
	if err != nil {
		res.FailWithMsg("删除广告失败", c)
		return
	}
	res.OKWithMsg(fmt.Sprintf("共删除%d个广告", count), c)
}
```



## 查询广告列表

**步骤：**

1. 进行参数绑定
2. 判断请求头中有没有携带`GVB_REFERER`字段
   - `GVB_REFERER`中有包含的`admin`说明请求是从后台系统来的，可以查询所有广告
   - 没有包含`admin`说明请求是从前台来的，只可查看启用的广告即`is_show`为`true`
3. 查询广告列表

> `gorm`的特性，当布尔类型的值为`false`时，会查询所有数据即`(true,false)`

```go
package advert_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/models"
	"gvb_server/service/common"
	"gvb_server/utils/res"
	"strings"
)

// AdvertListView 查询广告列表
func (AdvertApi) AdvertListView(c *gin.Context) {
	var cr models.PageInfo
	err := c.ShouldBindQuery(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	isShow := true
	referer := c.GetHeader("GVB_REFERER")
	if strings.Contains(referer, "admin") {
		isShow = false
	}
	list, count, err := common.CommList(models.AdvertModel{IsShow: isShow}, common.Option{
		PageInfo: cr,
		Debug:    true,
	})
	if err != nil {
		res.FailWithMsg("查询广告列表失败", c)
		return
	}
	res.OKWithList(list, count, c)
}
```

## 广告路由

```go
package routers

import "gvb_server/api"

func (r RouterGroup) AdvertRouter() {
	app := api.ApiGroupApp.AdvertApi
	r.GET("/advert", app.AdvertListView)
	r.POST("/advert", app.AdvertCreateView)
	r.PUT("/advert/:id", app.AdvertUpdateView)
	r.DELETE("/advert", app.AdvertRemoveView)
}
```

