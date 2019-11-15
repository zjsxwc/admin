## 代码阅读

#### 关于为什么很少用ORM外键已经相关的一系列级联删除、更新

1.因为数据的价值变大基本上都是通过标记字段来软删除，而不会直接删除或者级联覆盖

2.业务都在高级语言里面，而不在数据库里面

3.性能

4.方便后来者接手

说明：
Beego Orm不创建真正的数据库外键，但是带了软外键支持的，所以外键on_delete(`cascade,set_null,set_default,do_nothing`)对于orm model的删除仍旧有效，但直接raw sql删除就不会级联处理，因为没有真正数据库外键创建。

i find that beego orm not create real foreign key in the Database(Mysql, sqlite), 

but when we do delete the Model in beego orm golang code, it will do delete the other related Model(s).

so there is the soft foreign key constraints in its golang code.

see the references of method `deleteRels()` https://github.com/astaxie/beego/blob/v1.12.0/orm/db.go#L805

#### 如何对数据库表初始化

使用`"-syncdb"`参数进入`models.Syncdb()` beego/admin/admin.go:37

`createdb()`连接数据库后创建对应名字的数据库

然后`Connect()`连接这个新创建的对应名字的数据库

然后创建所有model的数据库表。

###### 如何创建所有model的数据库表

由于golang运行之前会执行所有依赖的文件的`init()`方法，我们的所有model文件里都有一个`init()`方法，来把model注册到包`orm`里的全局变量`modelCache`里（类型是 sync.RWMutex 读写互斥锁）


#### 如何读写session

详见文档 https://beego.me/docs/mvc/controller/session.md

所有Controller的基类`beego.Controller`的方法`userinfo := this.GetSession("userinfo")`

```

// SetSession puts value into session.
func (c *Controller) SetSession(name interface{}, value interface{}) {
	if c.CruSession == nil {
		c.StartSession()
	}
	c.CruSession.Set(name, value)
}
// GetSession gets value from session.
func (c *Controller) GetSession(name interface{}) interface{} {
	if c.CruSession == nil {
		c.StartSession()
	}
	return c.CruSession.Get(name)
}

// DelSession removes value from session.
func (c *Controller) DelSession(name interface{}) {
	if c.CruSession == nil {
		c.StartSession()
	}
	c.CruSession.Delete(name)
}
// StartSession starts session and load old session data info this controller.
func (c *Controller) StartSession() session.Store {
	if c.CruSession == nil {
		c.CruSession = c.Ctx.Input.CruSession
	}
	return c.CruSession
}
```


#### sessionId的读取顺序


先读取cookie里的 sessionId，没有就读取query参数里的 sessionId，再没有就读取http头里的 sessionId


```
// getSid retrieves session identifier from HTTP Request.
// First try to retrieve id by reading from cookie, session cookie name is configurable,
// if not exist, then retrieve id from querying parameters.
//
// error is not nil when there is anything wrong.
// sid is empty when need to generate a new session id
// otherwise return an valid session id.
func (manager *Manager) getSid(r *http.Request) (string, error) {
	cookie, errs := r.Cookie(manager.config.CookieName)
	// conf.CookieName = BConfig.WebConfig.Session.SessionName:  "beegosessionID",
	// BConfig.WebConfig.Session.SessionNameInHTTPHeader:      "Beegosessionid",
	if errs != nil || cookie.Value == "" {
		var sid string
		if manager.config.EnableSidInURLQuery {
			errs := r.ParseForm()
			if errs != nil {
				return "", errs
			}

			sid = r.FormValue(manager.config.CookieName)
		}

		// if not found in Cookie / param, then read it from request headers
		if manager.config.EnableSidInHTTPHeader && sid == "" {
			sids, isFound := r.Header[manager.config.SessionNameInHTTPHeader]
			if isFound && len(sids) != 0 {
				return sids[0], nil
			}
		}

		return sid, nil
	}

	// HTTP Request contains cookie for sessionid info.
	return url.QueryUnescape(cookie.Value)
}
```

#### 如何读取session

前提 `BConfig.WebConfig.Session.SessionOn == True`，

先在`context.Input.CruSession, err = GlobalSessions.SessionStart(rw, r)`里获取session id，如果有session id那么根据这个值获取对应的 `SessionStore`后直接用。

如果之前没有session id，就生成新session id与对应的session
```
github.com/astaxie/beego/router.go:715
	// GlobalSessions is the instance for the session manager
	GlobalSessions *session.Manager
	...
	// session init
	if BConfig.WebConfig.Session.SessionOn {
		var err error
		context.Input.CruSession, err = GlobalSessions.SessionStart(rw, r)


github.com/astaxie/beego/session/session.go:207
// SessionStart generate or read the session id from http request.
// if session id exists, return SessionStore with this id.
func (manager *Manager) SessionStart(w http.ResponseWriter, r *http.Request) (session Store, err error) {
	sid, errs := manager.getSid(r)
	if errs != nil {
		return nil, errs
	}

	if sid != "" && manager.provider.SessionExist(sid) {
		return manager.provider.SessionRead(sid)
	}

//  如果之前没有session id，就生成新session id  <----------------------------
	// Generate a new session
	sid, errs = manager.sessionID()
	if errs != nil {
		return nil, errs
	}

	session, err = manager.provider.SessionRead(sid)
	if err != nil {
		return nil, err
	}
	cookie := &http.Cookie{
		Name:     manager.config.CookieName,
		Value:    url.QueryEscape(sid),
		Path:     "/",
		HttpOnly: !manager.config.DisableHTTPOnly,
		Secure:   manager.isSecure(r),
		Domain:   manager.config.Domain,
	}
	if manager.config.CookieLifeTime > 0 {
		cookie.MaxAge = manager.config.CookieLifeTime
		cookie.Expires = time.Now().Add(time.Duration(manager.config.CookieLifeTime) * time.Second)
	}
	if manager.config.EnableSetCookie {
		http.SetCookie(w, cookie)
	}
	r.AddCookie(cookie)

	if manager.config.EnableSidInHTTPHeader {
		r.Header.Set(manager.config.SessionNameInHTTPHeader, sid)
		w.Header().Set(manager.config.SessionNameInHTTPHeader, sid)
	}

	return
}


//github.com/astaxie/beego/session/session.go:342
func (manager *Manager) sessionID() (string, error) {
	b := make([]byte, manager.config.SessionIDLength)
	n, err := rand.Read(b)
	if n != len(b) || err != nil {
		return "", fmt.Errorf("Could not successfully read from the system CSPRNG")
	}
	return manager.config.SessionIDPrefix + hex.EncodeToString(b), nil
}
```

如果不是内存存储session的话还要在main.go的init里配置gob
```

func init()  {
	//基础类型 int、string等默认已经注册了的，不需要为session注册，需要注册特定类型，在文件`sess_utils.go`里init注册了的也不用再注册
	gob.Register(models.User{})
	gob.Register(map[string]bool{})
	gob.Register(map[string]string{})
}
```


#### 如何判断权限

```
alist := make([]*AccessNode, 0)
type AccessNode struct {
	Id        int64
	Name      string
	Childrens []*AccessNode
}
```

```
	accesslist := make(map[string]bool)
	for _, v := range alist {
		for _, v1 := range v.Childrens {
			for _, v2 := range v1.Childrens {
				vname := strings.Split(v.Name, "/")
				v1name := strings.Split(v1.Name, "/")
				v2name := strings.Split(v2.Name, "/")
				str := fmt.Sprintf("%s/%s/%s", strings.ToLower(vname[0]), strings.ToLower(v1name[0]), strings.ToLower(v2name[0]))
				accesslist[str] = true
			}
		}
	}
```

accesslist值例子，就是通过一级到三级Node的Name字段拼key "%s/%s/%s"
```
map[rbac/group/index:true rbac/user/adduser:true rbac/user/index:true]
```

是否有权限就是先通过url获取key

```
params := strings.Split(strings.ToLower(strings.Split(ctx.Request.RequestURI, "?")[0]), "/")
// 请求 http://127.0.0.1:8080/rbac/user/index?xx=yy 就得到 params为 list["" "rbac" "user" "index"] 长度为4
//To test whether permissions
func AccessDecision(params []string, accesslist map[string]bool) bool {
	if CheckAccess(params) {
		s := fmt.Sprintf("%s/%s/%s", params[1], params[2], params[3])
		if len(accesslist) < 1 {
			return false
		}
		_, ok := accesslist[s]
		if ok != false {
			return true
		}
	} else {
		return true
	}
	return false
}

beego.InsertFilter("/*", beego.BeforeRouter, Check)//Check就是每次访问拦截鉴权
```

如果用户没有登录就跳去登录页面
```
rbac_auth_gateway := beego.AppConfig.String("rbac_auth_gateway")//app.conf 里 rbac_auth_gateway = /public/login
			if CheckAccess(params) {
				uinfo := ctx.Input.Session("userinfo")
				if uinfo == nil {
					ctx.Redirect(302, rbac_auth_gateway)
                    return
				}
```

如果当前登录用户访问的url不在其accesslist里面，那么返回json `{"status": false, "info": "权限不足"}`

## beego admin

基于beego，jquery easyui ,bootstrap的一个后台管理系统

VERSION = "0.1.1"

## 获取安装

执行以下命令，就能够在你的`GOPATH/src` 目录下发现beego admin
```bash
$ go get github.com/beego/admin
```

## 初次使用

### 创建应用
首先,使用bee工具创建一个应用程序，参考[`http://beego.me/quickstart`](beego的入门)
```
$ bee new hello
```
创建成功以后，你能得到一个名叫hello的应用程序，
现在开始可以使用它了。找到到刚刚新建的程序`hello/routers/router.go`这个文件
```go
import (
	"hello/controllers" 		//自身业务包
	"github.com/astaxie/beego"  //beego 包
	"github.com/beego/admin"  //admin 包
)

```
引入admin代码，再`init`函数中使用它
```go
func init() {
	admin.Run()
	beego.Router("/", &controllers.MainController{})
}
```
### 配置文件

数据库目前仅支持MySQL,PostgreSQL,sqlite3,后续会添加更多的数据库支持。

数据库的配置信息需要填写，程序会根据配置自动建库
MySQL数据库链接信息
```
db_host = localhost
db_port = 3306
db_user = root
db_pass = root
db_name = admin
db_type = mysql
```
postgresql数据库链接信息
```
db_host = localhost
db_port = 5432
db_user = postgres
db_pass = postgres
db_name = admin
db_type = postgres
db_sslmode=disable
```
sqlite3数据库链接信息
```
###db_path 是指数据库保存的路径，默认是在项目的根目录
db_path = ./
db_name = admin
db_type = sqlite3
```
把以上信息配置成你自己数据库的信息。

还有一部分权限系统需要配置的信息
```
sessionon = true
rbac_role_table = role
rbac_node_table = node
rbac_group_table = group
rbac_user_table = user
#admin用户名 此用户登录不用认证
rbac_admin_user = admin

#默认不需要认证模块
not_auth_package = public,static
#默认认证类型 0 不认证 1 登录认证 2 实时认证
user_auth_type = 1
#默认登录网关
rbac_auth_gateway = /public/login
#默认模版
template_type=easyui
```
以上配置信息都需要加入到hello/conf/app.conf文件中, 可以参考admin/conf/app.conf的配置。

### 复制静态文件

最后还需要把js，css，image，tpl这些文件复制过来。
```bash
$ cd $GOPATH/src/hello
$ cp -R ../github.com/beego/admin/static ./
$ cp -R ../github.com/beego/admin/views ./

```
### 编译项目

全部做好了以后。就可以编译了,进入hello目录
```
$ go build
```
首次启动需要创建数据库、初始化数据库表。
```bash
$ ./hello -syncdb
```
好了，现在可以通过浏览器地址访问了[`http://localhost:8080/`](http://localhost:8080/)

默认得用户名密码都是admin

