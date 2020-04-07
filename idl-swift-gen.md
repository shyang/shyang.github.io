# idl 自动生成 Swift 网络封装

Thrift IDL 原始格式类似如下，字段大量精简，只是参考。

```thrift
/** 用户注册 */
user.UserSignupResponse UserSignUp(1: user.UserSignupRequest req) (api.post="/carter/sign_up")

/** 注册 request */
struct UserSignupRequest {
    /** Lark邮箱 */
    1: string LarkEmail (api.body="lark_email")
}

/** 注册 response */
struct UserSignupResponse {
    /** 状态码, Required */
    1: i32 StatusCode (api.body="status_code")
    /** 状态信息, Required */
    2: general.StatusInfo StatusInfo (api.body="status_info, omitempty")
    /** 新创建的用户 id, Required */
    3: i64 CreatedUserID (api.body="created_user_id, string, omitempty")
}
```

先用官方工具转为 json，这样解析起来就极为省力。

```json
{
  "name": "UserSignUp",
  "returnTypeId": "struct",
  "returnType": {
    "typeId": "struct",
    "class": "UserSignupResponse"
  },
  "oneway": false,
  "doc": "用户注册\n",
  "annotations":           {
    "api.post": "\/carter\/sign_up"
  },
  "arguments": [
    {
      "key": 1,
      "name": "req",
      "typeId": "struct",
      "type": {
        "typeId": "struct",
        "class": "UserSignupRequest"
      },
      "required": "req_out"
    }
  ],
  "exceptions": [
  ]
}
```

通过脚本解析 json，输出 swift 代码，注释都可以一一复制过来。

```swift
/// 用户注册
/// - Parameter larkEmail: Lark邮箱
/// - Returns: `Observable<UserSignupResponse>`: 可直接 subscribe()/flatMap() 等
public static func postSignUp(larkEmail: String? = "") -> Observable<UserSignupResponse> {
    VVRequest<UserSignupResponse>()
        .url("/carter/sign_up")
        .addParam(key: "lark_email", value: larkEmail)
        .method(.post)
        .send()
}

/// 用户注册
/// - Parameter larkEmail: Lark邮箱
/// - Returns: `VVRequest<UserSignupResponse>`: 进一步定制后再 .send().subscribe()
public static func postSignUpRequest(larkEmail: String? = "") -> VVRequest<UserSignupResponse> {
    VVRequest<UserSignupResponse>()
        .url("/carter/sign_up")
        .addParam(key: "lark_email", value: larkEmail)
        .method(.post)
}
```

使用示例

```swift
// 多个 API 串行：如注册成功后，拉取 profile
Carter.postSignUp().flatMap { _ in
    Carter.getMe()
}.subscribe(onNext: { me in
    // 处理 me
}, onError: { err in
    // 处理 err
})


// 多个 API 并行，如进入艺人的主页同时获取其个人信息、专辑等
Observable.zip(
    Carter.getArtistDetail(id: userId),
    Carter.getArtistTracks(id: userId),
    Carter.getArtistAlbums(id: userId)
).subscribe(onNext: { (artist, tracks, albums) in
    // 处理拿到的三个数据
}, onError: { err in
    // 处理 err
})

```

未完成部分：其实数据对象也有清晰的 idl 描述，这部分也完全可以自动化转为代码，减少机械劳动。由于引入时大部分已经 model 已经定义完毕，只做了 typealias 桥接，暂时未对它们做代码生成。

新项目可以全套自动生成 Model+RPC。
