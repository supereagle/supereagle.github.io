---
layout:     post
title:      "Golang中HTTP Request的Auth认证"
subtitle:   "HTTP Request with Auth in Golang"
date:       2017-11-22
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
    - HTTP
---

绝大多数系统为了安全考虑，都会有认证机制，需要用户先登录，才能进行操作。如果是通过程序去调用这些系统的REST APIs，request请求中必须带有
Authorization信息，才能被这些系统接受和响应。

HTTP Request的Auth机制常用的主要有两种：
* Basic Auth：通过username和password来进行认证，是一种最简单的机制。
* Bearer Token：通过将Auth所需要的所有信息都加密编码到一串字符组成的token中，被访问系统受到请求后，解密token中信息来进行认证。这是一种比较复杂、但是又比较安全的机制。

如何在HTTP Client给request添加Auth信息，下面详细介绍两种实现途径。

## 自研HTTP Client

如果为某个系统从零开发一套HTTP Client，那么在每个request请求中添加auth信息就非常容易。因为，request请求是我们自己构建出来，如何发送
request请求也是完全由我们控制。只需要几行代码就可以搞定，示例代码如下：

```
func Do(method string, url string, payload io.Reader) (*http.Response, error) {
	req, err := http.NewRequest(method, url, payload)
	if err != nil {
		return nil, err
	}

	// Set the auth for the request.
	req.SetBasicAuth(username, password)

	return http.DefaultClient.Do(req)
}
```

像[gojenkins](https://github.com/bndr/gojenkins)就是为Jenkins开发一套Go语言的HTTP Client，它就是采用这种实现方式。
其他的像集成[Clair](https://github.com/coreos/clair)和[Docker Registry](https://github.com/docker/distribution)的镜像安全扫描工具[Klar](https://github.com/optiopay/klar)
也是采用这种方式访问带Auth的Docker Registry。

## 基于Proxy的HTTP Client

由于自研的HTTP Client灵活度很高，request请求中加auth也就是几行代码的事。但是，不可能为每个带认证的系统都自研一套HTTP Client，这样投入成本过高，
而且如果已经有开源的HTTP Client可以使用，那么自研基本上就没有多大意义。

如果某个系统已经有一套开源的HTTP Client可以拿来直接使用，是一件非常美好的事情，但是，并不一定完全满足我们的需求。例如，最近在使用Github的
Go语言的Client [go-github](https://github.com/google/go-github)的时候，就遇到给request加auth的问题。

```golang
// NewClient returns a new GitHub API client.  If a nil httpClient is
// provided, http.DefaultClient will be used.  To use API methods which require
// authentication, provide an http.Client that will perform the authentication
// for you (such as that provided by the golang.org/x/oauth2 library).
func NewClient(httpClient *http.Client) *Client {
	if httpClient == nil {
		httpClient = http.DefaultClient
	}
	baseURL, _ := url.Parse(defaultBaseURL)
	uploadURL, _ := url.Parse(uploadBaseURL)

	c := &Client{client: httpClient, BaseURL: baseURL, UserAgent: userAgent, UploadURL: uploadURL}
	c.common.client = c
	c.Activity = (*ActivityService)(&c.common)
	c.Admin = (*AdminService)(&c.common)
	c.Authorizations = (*AuthorizationsService)(&c.common)
	c.Gists = (*GistsService)(&c.common)
	c.Git = (*GitService)(&c.common)
	c.Gitignores = (*GitignoresService)(&c.common)
	c.Integrations = (*IntegrationsService)(&c.common)
	c.Issues = (*IssuesService)(&c.common)
	c.Licenses = (*LicensesService)(&c.common)
	c.Migrations = (*MigrationService)(&c.common)
	c.Organizations = (*OrganizationsService)(&c.common)
	c.Projects = (*ProjectsService)(&c.common)
	c.PullRequests = (*PullRequestsService)(&c.common)
	c.Reactions = (*ReactionsService)(&c.common)
	c.Repositories = (*RepositoriesService)(&c.common)
	c.Search = (*SearchService)(&c.common)
	c.Users = (*UsersService)(&c.common)
	return c
}
```

这个Client是封装用户提供的一个标准的HTTP Client，它只是帮忙构建和发送request请求，并处理response。注释中已经明确说了，authentication
相关的工作，交给用户提供的Client处理，该Client不负责。此时，用户需要提供一个自带auth功能的标准HTTP Client。

如果是采用Bearer Token的认证机制，可以使用官方提供的库[oauth2](https://github.com/golang/oauth2):

```golang
// Use token to new GitHub client.
tokenSource := oauth2.StaticTokenSource(
    &oauth2.Token{AccessToken: token},
)
client := oauth2.NewClient(oauth2.NoContext, tokenSource)
```

如果是采用Basic auth的认证机制，就需要用到一些技巧，充分利用HTTP Proxy的机制，就可以轻松为每个request加上Basic auth。

```golang
	// Proxy specifies a function to return a proxy for a given
	// Request. If the function returns a non-nil error, the
	// request is aborted with the provided error.
	// If Proxy is nil or returns a nil *URL, no proxy is used.
	Proxy func(*Request) (*url.URL, error)
```

Golang官方包`net/http`中已经对Proxy进行了详细的说明：该函数会为request返回一个proxy，如果返回非nil的error，request将被终止；如果返回
非nil的URL，proxy将不会生效。因此，可以充分利用这个特性，给每个request加上Basic auth，而又丝毫不影响request的行为。示例代码如下：

```golang
client := &http.Client{
    Transport: &http.Transport{
        Proxy: func(req *http.Request) (*url.URL, error) {
            req.SetBasicAuth(username, password)
            return nil, nil
        },
    },
}
```
