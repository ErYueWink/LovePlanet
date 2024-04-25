# 图片管理

## 图片上传

**步骤：**

1. 获取`MultipartForm`用于上传多个文件
2. 获取`Form`中的`images`文件
3. 判断文件上传的目录是否存在
4. 遍历上传的文件
5. 配置文件保存的位置，文件路径+文件名
6. 上传文件大小默认为字节，将字节转换为`MB`
7. 判断上传文件的大小是否大于预定的文件最大可上传的大小
   - 如果大于，返回错误信息
   - 否则继续向下执行
8. 保存文件
   - 保存失败，返回错误信息
9. 返回成功信息
10. 返回响应

```go
package images_api

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/utils/res"
	"io/fs"
	"os"
	"path"
)

type FileUploadResponse struct {
	FileName  string `json:"file_name"`
	IsSuccess bool   `json:"is_success"`
	Msg       string `json:"msg"`
}

// ImagesUploadView 上传单/多个文件
func (ImagesApi) ImagesUploadView(c *gin.Context) {
	// 获取MultipartForm
	form, err := c.MultipartForm()
	if err != nil {
		res.FailWithMsg("上传文件失败", c)
		return
	}
	// 获取form表单中的图片
	files, ok := form.File["images"]
	if !ok {
		res.FailWithMsg("上传图片失败", c)
		return
	}
	// 判断文件上传目录是否存在
	basePath := global.Config.Upload.Path
	_, err = os.ReadDir(basePath)
	if err != nil {
		// 不存在则创建目录
		err = os.MkdirAll(basePath, fs.ModePerm)
		if err != nil {
			global.Log.Error(err.Error())
			res.FailWithMsg("创建目录失败", c)
			return
		}
	}
	// 响应数据
	var fileUploadResponse []FileUploadResponse
	// 遍历上传的文件
	for _, file := range files {
		// 文件保存位置
		filePath := path.Join(basePath, file.Filename)
		// 计算上传的文件占几M
		size := float64(file.Size) / float64(1024*1024)
		// 如果上传文件的大小大于等于2M的话，上传失败
		if size >= float64(global.Config.Upload.Size) {
			global.Log.Error("上传文件失败")
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  file.Filename,
				IsSuccess: false,
				Msg:       fmt.Sprintf("上传文件大小为%.2fM,超过预定大小%dM", size, global.Config.Upload.Size),
			})
			continue // 开始第二次循环
		}
		err = c.SaveUploadedFile(file, filePath)
		if err != nil {
			global.Log.Error("保存文件失败")
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  file.Filename,
				IsSuccess: false,
				Msg:       "保存文件失败",
			})
			continue // 开启下一次循环
		}
		// 文件保存成功
		fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
			FileName:  filePath,
			IsSuccess: true,
			Msg:       "上传成功",
		})
	}
	res.OKWithData(fileUploadResponse, c)
}

```

## 图片白名单、黑名单

**黑名单：**用户上传的图片在黑名单列表中的话，拒绝上传

**白名单：**用户上传的图片在白名单列表中，上传成功

```go
var (
	// WhiteImageList 白名单
	WhiteImageList = []string{
		"jpg",
		"jpeg",
		"png",
		"gif",
		"tiff",
		"webp",
		"svg",
		"ico",
	}
)
```

**在`utils`包下编写工具类**

判断用户上传的图片是否在白名单列表中

```go
package utils

// arg:key 图片后缀名 list:图片后缀名白名单列表
func InList(key string, list []string) bool {
	for _, s := range list {
		if key == s {
			return true
		}
	}
	return false
}
```

**获取文件后缀名**

```go
// 获取文件后缀名
		fileName := file.Filename
		nameList := strings.Split(fileName, ".")
		suffix := strings.ToLower(nameList[len(nameList)-1])
		if !utils.InList(suffix, WhiteImageList) {
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  file.Filename,
				IsSuccess: false,
				Msg:       "文件非法",
			})
			continue
		}
```

## 图片入库

**在`utils`包下新建`MD5`加密的方法**

```go
package utils

import (
	"crypto/md5"
	"encoding/hex"
)

// Md5 md5
func Md5(src []byte) string {
	m := md5.New()
	m.Write(src)
	res := hex.EncodeToString(m.Sum(nil))
	return res
}
```

### 修改图片上传方法

**修改的地方**

1. 在图片保存之前根据图片`Hash`值判断用户上传的图片和数据库中保存图片的`Hash`值是否相同
   - 如果相同：图片重复开启新一次循环
   - 如果不同：继续向下执行
2. 在保存图片成功后，操作数据库图片入库

```go
package images_api

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/models"
	"gvb_server/utils"
	"gvb_server/utils/res"
	"io"
	"io/fs"
	"os"
	"path"
	"strings"
)

var (
	// WhiteImageList 白名单
	WhiteImageList = []string{
		"jpg",
		"jpeg",
		"png",
		"gif",
		"tiff",
		"webp",
		"svg",
		"ico",
	}
)

type FileUploadResponse struct {
	FileName  string `json:"file_name"`
	IsSuccess bool   `json:"is_success"`
	Msg       string `json:"msg"`
}

// ImagesUploadView 上传单/多个文件
func (ImagesApi) ImagesUploadView(c *gin.Context) {
	// 获取MultipartForm
	form, err := c.MultipartForm()
	if err != nil {
		res.FailWithMsg("上传文件失败", c)
		return
	}
	// 获取form表单中的图片
	files, ok := form.File["images"]
	if !ok {
		res.FailWithMsg("上传图片失败", c)
		return
	}
	// 判断文件上传目录是否存在
	basePath := global.Config.Upload.Path
	_, err = os.ReadDir(basePath)
	if err != nil {
		// 不存在则创建目录
		err = os.MkdirAll(basePath, fs.ModePerm)
		if err != nil {
			global.Log.Error(err.Error())
			res.FailWithMsg("创建目录失败", c)
			return
		}
	}
	// 响应数据
	var fileUploadResponse []FileUploadResponse
	// 遍历上传的文件
	for _, file := range files {
		// 获取文件后缀名
		fileName := file.Filename
		nameList := strings.Split(fileName, ".")
		suffix := strings.ToLower(nameList[len(nameList)-1])
		if !utils.InList(suffix, WhiteImageList) {
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  file.Filename,
				IsSuccess: false,
				Msg:       "文件非法",
			})
			continue
		}
		// 文件保存位置
		filePath := path.Join(basePath, file.Filename)
		// 计算上传的文件占几M
		size := float64(file.Size) / float64(1024*1024)
		// 如果上传文件的大小大于等于2M的话，上传失败
		if size >= float64(global.Config.Upload.Size) {
			global.Log.Error("上传文件失败")
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  file.Filename,
				IsSuccess: false,
				Msg:       fmt.Sprintf("上传文件大小为%.2fM,超过预定大小%dM", size, global.Config.Upload.Size),
			})
			continue // 开始第二次循环
		}
		fileObj, err := file.Open()
		if err != nil {
			global.Log.Error(err.Error())
		}
		fileBytes, err := io.ReadAll(fileObj)
		imageHash := utils.Md5(fileBytes)
		var bannerModel models.BannerModel
		// 判断图片是否重复
		err = global.DB.Take(&bannerModel, "hash = ?", imageHash).Error
		if err == nil {
			global.Log.Warning("图片重复")
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  bannerModel.Name,
				IsSuccess: false,
				Msg:       "图片重复",
			})
			continue
		}
		err = c.SaveUploadedFile(file, filePath)
		if err != nil {
			global.Log.Error("保存文件失败")
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  file.Filename,
				IsSuccess: false,
				Msg:       "保存文件失败",
			})
			continue // 开启下一次循环
		}
		// 文件保存成功
		fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
			FileName:  filePath,
			IsSuccess: true,
			Msg:       "上传成功",
		})
		// 图片入库
		global.DB.Create(&models.BannerModel{
			Name: fileName,
			Hash: imageHash,
			Path: filePath,
		})
	}
	res.OKWithData(fileUploadResponse, c)
}
```

## 图片列表

### 添加返回列表数据的方法

**修改`response.go`文件**

```go
type ListResponse[T any] struct {
	List  T     `json:"list"`
	Count int64 `json:"count"`
}
// OKWithList 返回列表数据
func OKWithList[T any](data T, count int64, c *gin.Context) {
	OKWithData(ListResponse[any]{
		Count: count,
		List:  data,
	}, c)
}
```

### **封装通用查询方法**

```go
package common

import (
	"gorm.io/gorm"
	"gvb_server/global"
	"gvb_server/models"
)

type Option struct {
	models.PageInfo
	Debug bool
}

// CommList 通用查询
func CommList[T any](model T, option Option) (list []T, count int64, err error) {
	DB := global.DB
	// Debug模式开启日志显示
	if option.Debug {
		DB = global.DB.Session(&gorm.Session{Logger: global.MysqlLog})
	}
	// 计算偏移量
	offset := (option.Page - 1) * option.Limit
	if offset < 0 {
		offset = 0
	}
	// 页数默认为10
	if option.Limit == 0 {
		option.Limit = 10
	}
	// 获取总条数
	count = DB.Select("id").Find(&list).RowsAffected
	// 分页查询
	err = DB.Limit(option.Limit).Offset(offset).Find(&list).Error
	return list, count, err
}
```

### 图片列表查询

```go
package images_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/models"
	"gvb_server/service/common"
	"gvb_server/utils/res"
)

// ImagesListView 查询图片列表
func (ImagesApi) ImagesListView(c *gin.Context) {
	var cr models.PageInfo
	err := c.ShouldBindQuery(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	list, count, err := common.CommList(models.BannerModel{}, common.Option{
		PageInfo: cr,
		Debug:    true,
	})
	if err != nil {
		res.FailWithMsg("查询图片列表失败", c)
		return
	}
	res.OKWithList(list, count, c)
}
```

## 图片删除

在`models`目录中的`enter.go`文件中新增以下内容

```go
type RemoveRequest struct {
	IDList []uint `json:"id_list"`
}
```

**图片删除方法**

```go
package images_api

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/models"
	"gvb_server/utils/res"
)

// ImagesRemoveApi 删除图片
func (ImagesApi) ImagesRemoveApi(c *gin.Context) {
	var cr models.RemoveRequest
	err := c.ShouldBindJSON(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	var bannerList []models.BannerModel
	count := global.DB.Select("id").Find(&bannerList, cr.IDList).RowsAffected
	if count == 0 {
		res.FailWithMsg("没有该图片", c)
		return
	}
	// 删除图片
	err = global.DB.Delete(&bannerList).Error
	if err != nil {
		res.FailWithMsg("删除图片失败", c)
		return
	}
	res.OKWithMsg(fmt.Sprintf("共删除%d张图片", count), c)
}
```

## 修改图片名称

**在工具类中新增参数校验的方法**

```go
package utils

import (
	"github.com/go-playground/validator/v10"
	"reflect"
)

// GetValidMsg 返回结构体中的msg参数
func GetValidMsg(err error, obj any) string {
	// 使用的时候，需要传obj的指针
	getObj := reflect.TypeOf(obj)
	// 将err接口断言为具体类型
	if errs, ok := err.(validator.ValidationErrors); ok {
		// 断言成功
		for _, e := range errs {
			// 循环每一个错误信息
			// 根据报错字段名，获取结构体的具体字段
			if f, exits := getObj.Elem().FieldByName(e.Field()); exits {
				msg := f.Tag.Get("msg")
				return msg
			}
		}
	}

	return err.Error()
}
```

**在`response`文件中新增参数绑定失败的方法**

```go
// FailWithError 参数绑定失败
func FailWithError(err error, obj any, c *gin.Context) {
	msg := utils.GetValidMsg(err, obj)
	FailWithMsg(msg, c)
}
```

**修改图片名称方法**

```go
package images_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/models"
	"gvb_server/utils/res"
)

type ImagesUpdateRequest struct {
	ID   uint   `json:"id" binding:"required" msg:"请输入图片ID"`
	Name string `json:"name" binding:"required" msg:"请输入图片名称"`
}

// ImagesUpdateView 编辑图片
func (ImagesApi) ImagesUpdateView(c *gin.Context) {
	var cr ImagesUpdateRequest
	err := c.ShouldBindJSON(&cr)
	if err != nil {
		res.FailWithError(err, &cr, c)
		return
	}
	var bannerModel models.BannerModel
	err = global.DB.Take(&bannerModel, cr.ID).Error
	if err != nil {
		res.FailWithMsg("图片不存在", c)
		return
	}
	// 图片存在修改图片
	err = global.DB.Model(&bannerModel).Update("name", cr.Name).Error
	if err != nil {
		res.FailWithMsg("图片修改失败", c)
		return
	}
	res.OKWithMsg("图片修改成功", c)
}
```

## 上传图片至七牛云

**在`plugins`目录下新增七牛云相关配置**

```go
package qiniu

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"github.com/qiniu/go-sdk/v7/auth/qbox"
	"github.com/qiniu/go-sdk/v7/storage"
	"gvb_server/config"
	"gvb_server/global"

	"time"
)

// 获取上传的token
func getToken(q config.QiNiu) string {
	accessKey := q.AccessKey
	secretKey := q.SecretKey
	bucket := q.Bucket
	putPolicy := storage.PutPolicy{
		Scope: bucket,
	}
	mac := qbox.NewMac(accessKey, secretKey)
	upToken := putPolicy.UploadToken(mac)
	return upToken
}

// 获取上传的配置
func getCfg(q config.QiNiu) storage.Config {
	cfg := storage.Config{}
	// 空间对应的机房
	zone, _ := storage.GetRegionByID(storage.RegionID(q.Zone))
	cfg.Zone = &zone
	// 是否使用https域名
	cfg.UseHTTPS = false
	// 上传是否使用CDN上传加速
	cfg.UseCdnDomains = false
	return cfg

}

// UploadImage 上传图片  文件数组，前缀
func UploadImage(data []byte, imageName string, prefix string) (filePath string, err error) {
	if !global.Config.QiNiu.Enable {
		return "", errors.New("请启用七牛云上传")
	}
	q := global.Config.QiNiu
	if q.AccessKey == "" || q.SecretKey == "" {
		return "", errors.New("请配置accessKey及secretKey")
	}
	if float64(len(data))/1024/1024 > q.Size {
		return "", errors.New("文件超过设定大小")
	}
	upToken := getToken(q)
	cfg := getCfg(q)

	formUploader := storage.NewFormUploader(&cfg)
	ret := storage.PutRet{}
	putExtra := storage.PutExtra{
		Params: map[string]string{},
	}
	dataLen := int64(len(data))

	// 获取当前时间
	now := time.Now().Format("20060102150405")
	key := fmt.Sprintf("%s/%s__%s", prefix, now, imageName)

	err = formUploader.Put(context.Background(), &ret, upToken, key, bytes.NewReader(data), dataLen, &putExtra)
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%s%s", q.CDN, ret.Key), nil

}
```

**修改图片上传，在图片上传之前判断七牛云存储是否打开**

启用七牛云存储新增如下代码：

```go
// 启用七牛云存储，就上传文件到七牛云
		if global.Config.QiNiu.Enable {
			qiniuFilePath, err := qiniu.UploadImage(fileBytes, fileName, "gvb")
			if err != nil {
				global.Log.Error(err.Error())
				continue
			}
			fileUploadResponse = append(fileUploadResponse, FileUploadResponse{
				FileName:  qiniuFilePath,
				IsSuccess: true,
				Msg:       "上传 文件至七牛云",
			})
			// 图片入库
			global.DB.Create(&models.BannerModel{
				Name:      fileName,
				Hash:      imageHash,
				Path:      filePath,
				ImageType: ctype.QINIU,
			})
			continue
		}
```

