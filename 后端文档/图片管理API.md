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

