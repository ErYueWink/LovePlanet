# 系统配置API

## 网站配置

在`core.conf`下新增修改`Yaml`文件的函数

**步骤：**

1. 将结构体转换为字节数组
2. 向`Yaml`文件中写入新数据

```go
// SetYaml 修改Yaml文件
func SetYaml() error {
	// 将结构体转换为字节数组
	bytes, err := yaml.Marshal(global.Config)
	if err != nil {
		return err
	}
	err = ioutil.WriteFile(yamlFile, bytes, fs.ModePerm)
	if err != nil {
		return err
	}
	return nil
}
```



### **查询网站详细信息**

```go
package settings_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/utils/res"
)

// SettingsSiteInfoView 获取网站信息
func (SettingsApi) SettingsSiteInfoView(c *gin.Context) {
	res.OKWithData(global.Config.SiteInfo, c)
}
```

### **修改网站信息**

```go
package settings_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/config"
	"gvb_server/core"
	"gvb_server/global"
	"gvb_server/utils/res"
)

// SettingsSiteUpdateInfoView 修改网站信息
func (SettingsApi) SettingsSiteUpdateInfoView(c *gin.Context) {
	var cr config.SiteInfo
	err := c.ShouldBindJSON(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	global.Config.SiteInfo = cr
	err = core.SetYaml() // 修改配置文件
	if err != nil {
		res.FailWithMsg(err.Error(), c)
		global.Log.Error(err) // 修改配置文件失败
		return
	}
	res.OKWith(c)
}
```

### **查询某一项的配置信息(QQ/Email/QiNiu/JWT)**

```go
package settings_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/global"
	"gvb_server/utils/res"
)

var (
	QQNAME    = "qq"
	JWTNAME   = "jwt"
	EMAILNAME = "email"
	QINIUNAME = "qiniu"
)

type SettingsUri struct {
	Name string `uri:"name" binding:"required"`
}

// SettingsInfoView 获取某一项的配置文件信息
func (SettingsApi) SettingsInfoView(c *gin.Context) {
	var cr SettingsUri
	err := c.ShouldBindUri(&cr)
	// 参数绑定失败
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	GetSettingInfo(cr.Name, c)
}

func GetSettingInfo(name string, c *gin.Context) {
	switch name {
	case QQNAME:
		res.OKWithData(global.Config.QQ, c)
	case JWTNAME:
		res.OKWithData(global.Config.Jwt, c)
	case EMAILNAME:
		res.OKWithData(global.Config.Email, c)
	case QINIUNAME:
		res.OKWithData(global.Config.QiNiu, c)
	default:
		res.FailWithMsg("没有该配置文件", c)
	}

}
```

### **修改某一项的配置信息**

```go
package settings_api

import (
	"github.com/gin-gonic/gin"
	"gvb_server/config"
	"gvb_server/core"
	"gvb_server/global"
	"gvb_server/utils/res"
)

// SettingsInfoUpdateView 修改某一项的配置文件信息
func (SettingsApi) SettingsInfoUpdateView(c *gin.Context) {
	var cr SettingsUri
	err := c.ShouldBindUri(&cr)
	if err != nil {
		res.FailErrorCode(res.ArgumentError, c)
		return
	}
	err = UpdateSettingInfo(cr.Name, c)
	if err != nil {
		res.FailWithMsg("修改配置文件失败", c)
		return
	}
	res.OKWithMsg("修改配置文件成功", c)
}

func UpdateSettingInfo(name string, c *gin.Context) error {
	switch name {
	case QQNAME:
		var qq config.QQ
		err := c.ShouldBindJSON(&qq)
		if err != nil {
			return err
		}
		global.Config.QQ = qq
		break
	case JWTNAME:
		var jwt config.Jwt
		err := c.ShouldBindJSON(&jwt)
		if err != nil {
			return err
		}
		global.Config.Jwt = jwt
		break
	case EMAILNAME:
		var email config.Email
		err := c.ShouldBindJSON(&email)
		if err != nil {
			return err
		}
		global.Config.Email = email
		break
	case QINIUNAME:
		var qiniu config.QiNiu
		err := c.ShouldBindJSON(&qiniu)
		if err != nil {
			return err
		}
		global.Config.QiNiu = qiniu
		break
	default:
		res.FailWithMsg("没有该配置文件", c)
	}
	err := core.SetYaml()
	if err != nil {
		res.FailWithMsg("没有该配置文件", c)
		global.Log.Error("修改配置文件失败")
		return err
	}
	return nil
}
```

