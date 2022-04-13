[toc]

# 文档说明

本文用于解释当前流体网络后端 `Python` 项目中部分写法的解释和一些引用的第三方包功能的使用。

> 本文以 `poros` 项目为例(下文不再具体说明)，本文写作时其代码结构如下：
>
> ```python
> .
> ├── admin_app.py
> ├── app.py  # 服务入口文件
> ├── apps  # 包含 app工厂和其他功能逻辑
> ├── celery_app.py  # 新建一个 celery app
> ├── common  # 为解决冲突新建的文件(未确定)
> ├── conf  # 配置文件
> ├── data  # 前端礼物列表
> ├── entity  # mongodb 的存储对象
> ├── errors  # 基础错误类自定义集合
> ├── female_logic  # 女嘉宾上麦奖励逻辑
> ├── ipython_config.py  # ipython 配置文件(疑似无效)
> ├── log_config.py  # 日志配置并初始化部分logger对象
> ├── logic  # 主要业务逻辑代码实现
> ├── manage.py  # 一个业务脚本，以ipython方式打印结果
> ├── redis_client.py  # 初始化redis实例
> ├── requirements.txt
> ├── restart.sh  # shell方式重启系统脚本
> ├── run  # 启动文件配置集合
> ├── schema  # orm 实现序列化的集合
> ├── setup  # docker配置及程序启动shell脚本
> ├── shell_script  # shell 脚本
> ├── tasks  # 异步任务集合
> ├── templates  # 后台前端文件
> ├── test  # 单元测试文件集合
> ├── tmp_scripts  # 临时脚本集合
> ├── util  # 通用工具集合
> └── views  # 对外接口集合
> ```

## 写法

### 应用创建

项目创建采用[应用工厂](https://dormousehole.readthedocs.io/en/latest/patterns/appfactories.html?highlight=create_app)的方式创建`app`, 并初始化应用的基本功能.

```python
# app.py
from apps import create_app

app = create_app()


# apps/__init__.py
def create_app(config_object='conf.settings'):
    """Create application factory, as explained here: http://flask.pocoo.org/docs/patterns/appfactories/."""
    app = Flask(__name__)
    app.config.from_object(config_object)
    configure_logger(app)

    filter_warning()
    register_extensions(app)
    register_sentry(app)
    register_json_encoder(app)
    # register_db(app)
    register_redis(app)
    register_blueprints(app)
    register_apispec(app)
    register_errorhandler(app)
    register_request_handle(app)
    # register_ab_test()
    return app
```

具体初始化功能如下：

- 导入配置文件

  导入文件方式为配置对象加载。讲配置文件名称作为对象名称导入

- 初始化日志级别

- 配置`warning`日志过滤器。忽略`warning`日志

- 初始化插件(设置缓存记录器为`redis`,设置四个不同业务场景的异步`celery`)

- 注册`sentry`日志监察模块

- 自定义应用`json`序列化器

- 连接`redis`

- 注册蓝图

- 注册`swagger api`管理模块

- 自定义错误处理模块

- 生命周期记录

另有`before_request`初始化参数位置统一放到`request.all_param`中

> 仅`json`参数和`args`参数

还有`after_request`统一打印每一个请求的`ip`地址、具体路径、访问方式、参数内容、返回结果、返回状态等数据。

> `swagger` 和 `flask-apispec`访问数据不用打印详细日志

### 接口编写与返回

接口编写采用函数视图，一般辅之以一定的装饰器。

常用的装饰器有作为视图函数的路由设定，参数校验，返回数据限定与格式化，和登录认证四种。



接口业务逻辑内容以 `mongodb` 和 `redis` 互相配合，加之一定的业务逻辑为主。



以`apps/couple/get_newr_female_list`函数为例。截取部分代码如下：

```python
@couple_app.route('/near_female/list', methods=['POST'])  # 路由设置方式
@use_kwargs({'only_voice_room': fields.Bool(required=True, description='是否开黑房')})  # 必要参数设置与注释
@marshal_with(NearFemaleSchema(many=True), apply=False)  # 返回数据格式
@login_required()  # 限定登录用户
def get_near_female_list(only_voice_room):
    """获取熟人列表"""
    user = request.user  # 限定登录用户接口会由 login_required 装饰器根据用户的 token 查询用户信息并设置对应信息到 request.user 中
    if not NearListTest.is_test_group(user.user_id):
        return response_util.response(response_util.RetCodeAndMessage.Success, data=[])

    # 向 mongodb 查询数据
    rooms = CoupleRoom.objects_not_delete(
        is_online=True, room_type__in=(CoupleRoom.CoupleRoomType.DEFAULT, CoupleRoom.CoupleRoomType.FIVE_ROOM))
    if only_voice_room:
        rooms = rooms(only_voice_room=True, voice_chatting=None)
    else:
        rooms = rooms(only_voice_room__ne=True, voice_chatting=None)

    user_ip = request.headers.get("X-Real-Ip", "")
    test_couple_room_user = rds.smembers('test_couple_room_user')
    near_female_map = NearScoreSvc.get_near_map(user.user_id, score=15)
    data_list = []
    for room in rooms:
        # 测试房间仅公司ip or 测试白名单 显示
        if (IsCoupleRoomWhite.is_white_account(room.room_id)
                and user_ip != constants.OFFICE_IP
                and user.user_id not in test_couple_room_user):
            continue

        on_mic_user_map = room.get_room_on_mic_user_map()
        has_male = on_mic_user_map.get(CoupleRoom.MicPosition.MIC1)
        if room.room_type == CoupleRoom.CoupleRoomType.FIVE_ROOM:
            has_male = has_male and on_mic_user_map.get(CoupleRoom.MicPosition.MIC3)
        for position, mic_info in on_mic_user_map.items():
            if position not in (CoupleRoom.MicPosition.ANCHOR,
                                CoupleRoom.MicPosition.MIC2, CoupleRoom.MicPosition.MIC4):
                continue
            female_id = mic_info['user_id']
            # 跳过非熟人房间
            if female_id not in near_female_map:
                continue

            data_list.append({
                'room_id': room.room_id,
                'im_room_id': room.im_room_id,
                'female_uid': female_id,
                'status_str': '开黑中' if room.voice_chatting else '正在找CP',
                'has_male': bool(has_male),
                'room_type': room.room_type,
                'near_score': int(near_female_map.get(female_id)),
                'mic_pos': mic_info.get('position', -1)
            })

    # 排序 取前20个
    data_list.sort(key=lambda x: (x['has_male'], -near_female_map.get(x['female_uid'], 0)))
    data_list = data_list[:20]
    fill_user_info(data_list, user_id_key='female_uid')
    
    # 自定义response处理器，可缓存接口数据，添加部分参数(im_video_token)
    return response_util.response(response_util.RetCodeAndMessage.Success, data=data_list)
```

> - 接口的装饰器数量和必要性应根据实际接口请求为准。
> - 返回结果也并不一定完全采用自定义的 `response_util.response`.(如 `poros/views/gift/gift_list_info` 视图函数)

## 包

此项目采用主要存储方式为 `mongodb`，并采用了`restful`形式的`api`管理。在代码中涉及到了较多的包的使用。

这里附上部分包的介绍说明和使用方式

### `marshmallow`

`marshmallow`是一个用来将复杂的`orm`对象与`python`原生数据类型之间相互转换的库，简而言之，就是实现`object -> dict`， `objects -> list`, `string -> dict` 和 `string -> list`。

> 序列化：序列化的意思是将数据对象转化为可存储或可传输的数据类型 反序列化：将可存储或可传输的数据类型转化为数据对象
> 要进行序列化或反序列化，首先我们需要一个用来操作的object，这里我们先定义一个类：

```python
import datetime as dt


class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
        self.created_time = dt.datetime.now()
        
    def __repr__(self):
        return "<User(name={self.name!r})>".format(self=self)
```

#### 1、Scheme

要对一个类或者一个json数据实现相互转换(即序列化和反序列化), 需要一个中间载体, 这个载体就是Schema，另外Schema还可以用来做数据验证。

```java
# 这是一个简单的Scheme
from marshmallow import Schema, fields


class UserSchema(Schema):
    name = fields.String()
    email = fields.Email()
    created_time = fields.DateTime()
```

#### 2、Serializing(序列化)

使用scheme的dump()方法来序列化对象，返回的是dict格式的数据

另外schema的dumps()方法序列化对象，返回的是json编码格式的字符串。

```java
user = User("lhh","2432783449@qq.com")
schema = UserSchema()
res = schema.dump(user)
print(res)
# {'email': '2432783449@qq.com', 'created_time': '2021-05-28 20:43:08.946112', 'name': 'lhh'}  dict

res2 = schema.dumps(user)
print(res2)
# '{"name": "lhh", "email": "2432783449@qq.com", "created_time": "2021-05-28 20:45:17.418739"}'  json
```

#### 3、过滤输出

当不需要输出所有的字段时，可以在实例化Scheme时，声明only参数，来指定输出：

```java
summary_schema = UserSchema(only={"name","email"})
res = summary_schema.dump(user)
print(res)
# {"name": "lhh", "email": "2432783449@qq.com"}
```

#### 4、Deserializing(反序列化)

schema的load()方法与dump()方法相反，用于dict类型的反序列化。他将输入的字典格式数据转换成应用层数据结构。他也能起到验证输入的字典格式数据的作用。 同样，也有对json解码的loads()方法。用于string类型的反序列化。 默认情况下，load()方法返回一个字典，当输入的数据的值不匹配字段类型时，抛出 ValidationError 异常。

```java
user_data = {
    "name": "lhh",
    "email": "2432783449@qq.com",
    "created_time": "2021-05-28 20:45:17.418739"
}
schema = UserSchema()
res = schema.load(user_data)
print(res)
# {'created_time': '2021-05-28 20:45:17.418739', 'email': '2432783449@qq.com', 'name': 'lhh'}
```

对反序列化而言, 将传入的dict变成object更加有意义. 在Marshmallow中, dict -> object的方法需要自己实现, 然后在该方法前面加上一个装饰器post_load即可

```java
class UserSchema(Schema):
    name = fields.String()
    email = fields.Email()
    created_time = fields.DateTime()

    @post_load
    def make_user(self, data):
        return User(**data)
```

这样每次调用load()方法时, 会按照make_user的逻辑, 返回一个User类对象。

```java
user_data = {
    "name": "lhh",
    "email": "2432783449@qq.com"
}

schema = UserSchema()
res = schema.load(user_data)
print(res)
# <__main__.User object at 0x0000027BE9678128>
user = res
print("name: {}    email: {}".format(user.name, user.email))
# name: lhh    email: 2432783449@qq.com
```

#### 5、处理多个对象的集合

多个对象的集合如果是可迭代的，那么也可以直接对这个集合进行序列化或者反序列化。在实例化Scheme类时设置参数many=True

也可以不在实例化类的时候设置，而在调用dump()方法的时候传入这个参数。

```java
user1 = User(name="lhh1", email="2432783449@qq.com")
user2 = User(name="lhh2", email="2432783449@qq.com")
users = [user1, user2]

# 第一种方法
schema = UserSchema(many=True)
res = schema.dump(users)
print(res)

# 第二种方法
schema = UserSchema()
res = schema.dump(users,many=True)
print(res)
    
# [{'name': u'Mick',
#   'email': u'mick@stones.com',
#   'created_at': '2014-08-17T14:58:57.600623+00:00'}
#  {'name': u'Keith',
#   'email': u'keith@stones.com',
#   'created_at': '2014-08-17T14:58:57.600623+00:00'}]
```

#### 6、Validation(验证)

当不合法的数据通过Schema.load()或者Schema.loads()时，会抛出一个 ValidationError 异常。ValidationError.messages属性有验证错误信息，验证通过的数据在 ValidationError.valid_data 属性中 我们捕获这个异常，然后做异常处理。首先需要导入ValidationError这个异常

```java
from marshmallow import Schema,fields,ValidationError


class UserSchema(Schema):
    name = fields.String()
    email = fields.Email()
    created_time = fields.DateTime()

try:
    res = UserSchema().load({"name":"lhh","email":"lhh"})

except ValidationError as e:
    print(f"错误信息：{e.messages}  合法数据:{e.valid_data}")

'''
    当验证一个数据集合的时候，返回的错误信息会以 错误序号-错误信息 的键值对形式保存在errors中
'''
user_data = [
    {'email': '2432783449@qq.com', 'name': 'lhh'},
    {'email': 'invalid', 'name': 'Invalid'},
    {'name': 'wcy'},
    {'email': '2432783449@qq.com'},
]


try:
    schema = UserSchema(many=True)
    res = schema.load(user_data)
    print(res)
except ValidationError as e:
    print("错误信息：{}   合法数据：{}".format(e.messages, e.valid_data))
```

可以看到上面，有错误信息，但是对于没有传入的属性则没有检查，也就是说没有规定属性必须传入。

在Schema里规定不可缺省字段：设置参数required=True

> ```
> 可以看到上面，有错误信息，但是对于没有传入的属性则没有检查，也就是说没有规定属性必须传入。 在Schema里规定不可缺省字段：设置参数required=True
> ```

##### 6.1 自定义验证信息

在编写Schema类的时候，可以向内建的fields中设置validate参数的值来定制验证的逻辑, validate的值可以是函数, 匿名函数lambda, 或者是定义了**call**的对象。

```python
from marshmallow import Schema,fields,ValidationError


class UserSchema(Schema):
    name = fields.String(required=True, validate=lambda s:len(s) < 6)
    email = fields.Email()
    created_time = fields.DateTime()

user_data = {"name":"InvalidName","email":"2432783449@qq.com"}
try:
    res = UserSchema().load(user_data)
except ValidationError as e:
    print(e.messages)
```

**在验证函数中自定义异常信息：**

```python
#encoding=utf-8
from marshmallow import Schema,fields,ValidationError

def validate_name(name):
    if len(name) <=2:
        raise ValidationError("name长度必须大于2位")
    if len(name) >= 6:
        raise ValidationError("name长度不能大于6位")


class UserSchema(Schema):
    name = fields.String(required=True, validate=validate_name)
    email = fields.Email()
    created_time = fields.DateTime()

user_data = {"name":"InvalidName","email":"2432783449@qq.com"}
try:
    res = UserSchema().load(user_data)
except ValidationError as e:
    print(e.messages)
```

`注意`：只会在反序列化的时候发生验证！序列化的时候不会验证！

##### 6.2 将验证函数写在Schema中变成验证方法

在Schema中，使用validates装饰器就可以注册验证方法。

```python
#encoding=utf-8
from marshmallow import Schema, fields, ValidationError, validates


class UserSchema(Schema):
    name = fields.String(required=True)
    email = fields.Email()
    created_time = fields.DateTime()

    @validates("name")
    def validate_name(self, value):
        if len(value) <= 2:
            raise ValidationError("name长度必须大于2位")
        if len(value) >= 6:
            raise ValidationError("name长度不能大于6位")


user_data = {"name":"InvalidName","email":"2432783449@qq.com"}
try:
    res = UserSchema().load(user_data)
except ValidationError as e:
    print(e.messages)
```

##### 6.3 Required Fields(必填选项)

上面已经简单使用过required参数了。这里再简单介绍一下。

**自定义required异常信息：**

首先我们可以自定义在requird=True时缺失字段时抛出的异常信息：设置参数error_messages的值

```python
#encoding=utf-8
from marshmallow import Schema, fields, ValidationError, validates


class UserSchema(Schema):
    name = fields.String(required=True, error_messages={"required":"name字段必须的"})
    email = fields.Email()
    created_time = fields.DateTime()

    @validates("name")
    def validate_name(self, value):
        if len(value) <= 2:
            raise ValidationError("name长度必须大于2位")
        if len(value) >= 6:
            raise ValidationError("name长度不能大于6位")


user_data = {"email":"2432783449@qq.com"}
try:
    res = UserSchema().load(user_data)
except ValidationError as e:
    print(e.messages)
```

**Partial Loading忽略验证：**

使用required之后我们还是可以在传入数据的时候忽略这个必填字段。

```java
#encoding=utf-8
from marshmallow import Schema, fields, ValidationError, validates


class UserSchema(Schema):
    name = fields.String(required=True)
    age = fields.Integer(required=True)

# 方法一：在load()方法设置partial参数的值(元组)，表时忽略那些字段。
schema = UserSchema()
res = schema.load({"age": 42}, partial=("name",))
print(res)
# {'age': 42}

# 方法二：直接设置partial=True
schema = UserSchema()
res = schema.load({"age": 42}, partial=True)
print(res)
# {'age': 42}
```

看起来两种方法是一样的，但是方法一和方法二有区别：方法一只忽略传入partial的字段，方法二会忽略除前面传入的数据里已有的字段之外的所有字段

##### 6.4 指定默认值

`load_default`指定默认的反序列化值

`dump_default`指定默认的序列化值

```python
class UserSchema(Schema):
    id = fields.UUID(load_default=uuid.uuid1)
    birthdate = fields.DateTime(dump_default=dt.datetime(2017, 9, 29))


UserSchema().load({})
# {'id': UUID('337d946c-32cd-11e8-b475-0022192ed31b')}
UserSchema().dump({})
# {'birthdate': '2017-09-29T00:00:00+00:00'}
```

##### 6.5 对未知字段的处理

默认情况下，如果传入了未知的字段(Schema里没有的字段)，执行load()方法会抛出一个 ValidationError 异常。这种行为可以通过更改 unknown 选项来修改。

unknown 有三个值：

- EXCLUDE: exclude unknown fields(直接扔掉未知字段)
- INCLUDE: accept and include the unknown fields(接受未知字段)
- RAISE: raise a ValidationError if there are any unknown fields(抛出异常)

我们可以看到，默认的行为就是RAISE。有两种方法去更改：

方法一：在编写Schema类的时候在class Meta里修改

```python
from marshmallow import EXCLUDE,Schema,fields

class UserSchema(Schema):
    name = fields.String(required=True,error_messages={"required": "name字段必须填写"})
    email = fields.Email()
    created_time = fields.DateTime()


    class Meta:
        unknown  = EXCLUDE
```

方法二：在实例化Schema类的时候设置参数unknown的值

```python
class UserSchema(Schema):
    name = fields.Str(required=True, error_messages={"required": "name字段必须填写"})
    email = fields.Email()
    created_time = fields.DateTime()

shema = UserSchema(unknown=EXCLUDE)
```

#### 7、Schema.validate(校验数据)

如果只是想用Schema去验证数据, 而不进行反序列化生成对象, 可以使用Schema.validate() 可以看到, 通过schema.validate()会自动对数据进行校验, 如果有错误, 则会返回错误信息的dict,没有错误则返回空的dict，通过返回的数据, 我们就可以确认验证是否通过.

```python
#encoding=utf-8
from marshmallow import Schema,fields,ValidationError

class UserSchema(Schema):
    name = fields.Str(required=True, error_messages={"required": "name字段必须填写"})
    email = fields.Email()
    created_time = fields.DateTime()

user = {"name":"lhh","email":"2432783449"}
schema = UserSchema()
res = schema.validate(user)
print(res)  # {'email': ['Not a valid email address.']}

user = {"name":"lhh","email":"2432783449@qq.com"}
schema = UserSchema()
res = schema.validate(user)
print(res)  # {}
```

#### 8. Specifying Serialization/Deserialization Keys(指定序列化/反序列化键)

##### 8.1 Specifying Attribute Names(序列化时指定object属性对应fields字段)

Schema默认会序列化传入对象和自身定义的fields相同的属性, 然而你也会有需求使用不同的fields和属性名. 在这种情况下, 你需要明确定义这个fields将从什么属性名取值

```python
from marshmallow import fields,Schema,ValidationError
import datetime as dt

class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
        self.created_time = dt.datetime.now()


class UserSchema(Schema):
    full_name = fields.String(attribute="name")
    email_address = fields.Email(attribute="email")
    created_at = fields.DateTime(attribute="created_time")


user = User("lhh",email="2432783449@qq.com")
schema = UserSchema()
res = schema.dump(user)
print(res)
# {'email_address': '2432783449@qq.com', 'full_name': 'lhh', 'created_at': '2021-05-29T09:24:38.186191'}
```

如上所示：UserSchema中的full_name，email_address，created_at分别从User对象的name，email，created_time属性取值。

##### 8.2 反序列化时指定fields字段对应object属性

这个与上面相反，Schema默认反序列化传入字典和输出字典中相同的字段名. 如果你觉得数据不匹配你的schema, 可以传入load_from参数指定需要增加load的字段名(原字段名也能load, 且优先load原字段名)

```python
from marshmallow import fields,Schema,ValidationError
import datetime as dt

class UserSchema(Schema):
    full_name = fields.String(load_from="name")
    email_address = fields.Email(load_from="email")
    created_at = fields.DateTime(load_from="created_time")

user = {"full_name":"lhh","email_address":"2432783449@qq.com"}
schema = UserSchema()
res = schema.load(user)
print(res)
# {'full_name': 'lhh', 'email_address': '2432783449@qq.com'}
```

##### 8.3 让key同时满足序列化与反序列化的方法

```python
#encoding=utf-8
from marshmallow import fields,ValidationError,Schema

class UserSchema(Schema):
    full_name = fields.String(data_key="name")
    email_address = fields.Email(data_key="email")
    created_at = fields.DateTime(data_key="created_time")

# 序列化
user = {"full_name": "lhh", "email_address": "2432783449@qq.com"}
schema = UserSchema()
res = schema.dump(user)
print(res)
# {'name': 'lhh', 'email': '2432783449@qq.com'}


# 反序列化
user = {'name': 'lhh', 'email': '2432783449@qq.com'}
schema = UserSchema()
res = schema.load(user)
print(res)
# {'full_name': 'lhh', 'email_address': '2432783449@qq.com'}
```

#### 9. 重构：创建隐式字段

当Schema具有许多属性时，为每个属性指定字段类型可能会重复，特别是当许多属性已经是本地python的数据类型时。class Meta允许指定要序列化的属性，marshmallow将根据属性的类型选择适当的字段类型。

```python
# 重构Schema
class UserSchema(Schema):
    uppername = fields.Function(lambda obj: obj.name.upper())

    class Meta:
        fields = ("name", "email", "created_at", "uppername")
```

以上代码中， name将自动被格式化为String类型，created_at将被格式化为DateTime类型。

如果您希望指定除了显式声明的字段之外还包括哪些字段名，则可以使用附加选项。如下：

```python
class UserSchema(Schema):
    uppername = fields.Function(lambda obj: obj.name.upper())

    class Meta:
        # No need to include 'uppername'
        additional = ("name", "email", "created_at")
```

#### 10. 排序

对于某些用例，维护序列化输出的字段顺序可能很有用。要启用排序，请将ordered选项设置为true。这将指示marshmallow将数据序列化到`collections.OrderedDict`

```python
from collections import OrderedDict
import datetime as dt
from marshmallow import fields,ValidationError,Schema

class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
        self.created_time = dt.datetime.now()

class UserSchema(Schema):
    uppername = fields.Function(lambda obj: obj.name.upper())

    class Meta:
        fields = ("name", "email", "created_time", "uppername")
        ordered = True


user = User("lhh", "2432783449@qq.com")
schema = UserSchema()
res = schema.dump(user)
print(isinstance(res,OrderedDict))  # 判断变量类型
# True
print(res)
# OrderedDict([('name', 'lhh'), ('email', '2432783449@qq.com'), ('created_time', '2021-05-29T09:40:46.351382'), ('uppername', 'LHH')])
```

#### 11. “只读”与“只写”字段

在Web API的上下文中，序列化参数dump_only和反序列化参数load_only在概念上分别等同于只读和只写字段。

```python
from marshmallow import Schema,fields


class UserSchema(Schema):
    name = fields.Str()
    password = fields.Str(load_only=True)  # 等于只写
    created_at = fields.DateTime(dump_only=True)  # 等于只读
```

load时，dump_only字段被视为未知字段。如果unknown选项设置为include，则与这些字段对应的键的值将因此loaded而不进行验证。

#### 12. 序列化/反序列化时指定字段的默认值

序列化时输入值缺失用default指定默认值。反序列化时输入值缺失用missing指定默认值。

```python
#encoding=utf-8
import uuid
import datetime as dt
from marshmallow import fields,ValidationError,Schema


class UserSchema(Schema):
    id = fields.UUID(missing=uuid.uuid1)
    birthday = fields.DateTime(default=dt.datetime(1996,11,17))

# 序列化
res = UserSchema().dump({})
print(res)
# {'birthday': '1996-11-17T00:00:00'}

# 反序列化
res = UserSchema().load({'birthday': '1996-11-17T00:00:00'})
print(res)
# {'id': UUID('751d95db-c020-11eb-83eb-001a7dda7115'), 'birthday': datetime.datetime(1996, 11, 17, 0, 0)}
```

#### 13. 后续扩展

```python
from marshmallow import Schema, fields


class String128(fields.String):
    """
    长度为128的字符串类型
    """

    default_error_messages = {
        "type": "该字段只能是字符串类型",
        "invalid": "该字符串长度必须大于6",
    }

    def _deserialize(self, value, attr, data, **kwargs):
        if not isinstance(value, str):
            self.fail("type")
        if len(value) < 6:
            self.fail("invalid")


class AppSchema(Schema):
    name = String128(required=True)
    priority = fields.Integer()
    obj_type = String128()
    link = String128()
    deploy = fields.Dict()
    description = fields.String()
    projects = fields.List(cls_or_instance=fields.Dict)


app = {
    "name": "app11",
    "priority": 2,
    "obj_type": "web",
    "link": "123.123.00.2",
    "deploy": {"deploy1": "deploy1", "deploy2": "deploy2"},
    "description": "app111 test111",
    "projects": [{"id": 2}]
}

schema = AppSchema()
res = schema.validate(app)
print(res)
# {'obj_type': ['该字符串长度必须大于6'], 'name': ['该字符串长度必须大于6']}
```

> [marshmallow: simplified object serialization — marshmallow 3.15.0 documentation](https://marshmallow.readthedocs.io/en/stable/)

### `webargs`

`webargs`是用来参数解析和校验参数的工具。

#### 使用示例

```python
from flask import Flask
from webargs import fields
from webargs.flaskparser import use_args

app = Flask(__name__)


@app.route("/")
@use_args({"name": fields.Str(required=True)}, location="query")
def index(args):
    return "Hello " + args["name"]


if __name__ == "__main__":
    app.run()

# curl http://localhost:5000/\?name\='World'
# Hello World
```

默认情况下，`webargs`会自动解析`json body`数据。但是也支持其他类型。需要手动指定数据来源，设置方式为`location=xxx`("form", "query","json","headers", "cookies", "files", "paths")

> 更多使用方式及功能支持可参见官方文档：[webargs 8.1.0 documentation](https://webargs.readthedocs.io/en/latest/)

### `flask-apispec`

`flask-apispec` 是一款轻量级能自动生成`REST APIs`的`Flask`工具。使用了`webargs`做`request`解析，`marshmallow`做`response`格式化，和`apispec`自动生成`Swagger`。

#### 下载安装

```python
pip install flask-apispec
```

#### 使用实例

```python
from flask import Flask
from flask_apispec import use_kwargs, marshal_with

from marshmallow import Schema
from webargs import fields

from .models import Pet

app = Flask(__name__)

class PetSchema(Schema):
    class Meta:
        fields = ('name', 'category', 'size')

@app.route('/pets')
@use_kwargs({'category': fields.Str(), 'size': fields.Str()})
@marshal_with(PetSchema(many=True))
def get_pets(**kwargs):
    return Pet.query.filter_by(**kwargs)
```

#### 项目中使用实例

```python

# apps/__init__.py
def register_apispec(app):
    from apispec import APISpec
    from apispec.ext.marshmallow import MarshmallowPlugin
    from util.flask_apispec import FlaskPlugin, FlaskApiSpec
    from webargs.flaskparser import FlaskParser
    from marshmallow import EXCLUDE

    # 设置json参数校验时未知参数处理方式为丢弃未知字段
    class Parser(FlaskParser):
        DEFAULT_UNKNOWN_BY_LOCATION = {"json": EXCLUDE}

    app.config.update({
        'APISPEC_SPEC': APISpec(
            title="POROS",
            version="1.0.0",
            openapi_version="2.0",
            info=dict(description="POROS API"),
            plugins=[FlaskPlugin(), MarshmallowPlugin()],
        ),
        'APISPEC_FORMAT_RESPONSE': None,
        'APISPEC_WEBARGS_PARSER': Parser(),  # 启用指定解析器
    })
    docs = FlaskApiSpec(app, document_options=False)
    docs.register_existing_resources()
```



> 官方文档:[flask-apispec: Auto-documenting REST APIs for Flask — flask-apispec 0.7.0 documentation](https://flask-apispec.readthedocs.io/en/latest/index.html) 
>
> 源码：[jmcarp/flask-apispec (github.com)](https://github.com/jmcarp/flask-apispec)

### `mongoengine`

`MongoEngine`是一个基于`pymongo`开发的`ODM`库，对应与`SQLAlchemy`。同时，在`MongoEngine`基础上封装了`Flask-MongoEngine`，用于支持`flask`框架。

#### 1 连接数据库

##### 1.1 多种连接方式

```python
from mongoengine import connect

# 方法一：本地连接
connect('dbname', alias='别名')

# 方法二：远程连接
connect('dbname', host='远程连接地址', post=端口号)

# 方法三：连接带有验证的远程数据库
connect('dbname', username='用户名', password='密码', authentication_source='admin',  host='远程服务器IP地址', post=开放的端口号))
```

##### 1.2 连接多个数据库

要使用多个数据库，您可以使用connect()并为连接提供别名(alisa)，默认使用“default”。
在后台，这用于register_connection()存储数据，如果需要，您可以预先注册所有别名。

- 在不同数据库中定义文档

  通过在元数据中提供db_alias，可以将各个文档附加到不同的数据库 。这允许DBRef 对象指向数据库和集合。下面是一个示例模式，使用3个不同的数据库来存储数据

  ```python
  # 默认本地连接
  connect (alias = 'user-db-alias' ， db = 'user-db' )
  connect (alias = 'book-db-alias' ， db = 'book-db' )
  connect (alias = 'users-books-db -alias' ， db = 'users-books-db' )
  
  class  User (Document )：
      name  =  StringField ()
      meta  =  { 'db_alias' ： 'user-db-alias' }   # 指定数据对应的数据库
  
  class  Book(Document )：
      name  =  StringField ()
      meta  =  { 'db_alias' ： 'book-db-alias' } 
  
  class  AuthorBooks (Document )：
      author  =  ReferenceField (User )
      book  =  ReferenceField (Book )
      meta  =  { 'db_alias' ： ' users-books-db-alias' }
  ```

- 断开现有连接

  `disconnect`可用于断开特定连接。

  ```python
  from mongoengine import connect, disconnect
  connect('a_db', alias='db1')    # ==》 建立别名为“db1”的连接
  
  class User(Document):
      name = StringField()
      meta = {'db_alias': 'db1'}
  
  disconnect(alias='db1')   # ==》 断开 别名为“db1”的连接
  
  connect('another_db', alias='db1')      # ==》 由于上一步断开了别名为db1的连接，现在连接其它数据库时又可以使用别名“db1”作为连接名
  ```

##### 1.3 上下文管理器

有时您可能希望切换数据库或集合以进行查询

- 切换数据库

  `switch_db`上允许更改数据库别名给定类，允许快速和方便地跨数据库访问：

  > 切换数据库，必须预先注册别名(使用已注册的别名)

  ```python
  from mongoengine.context_managers import switch_db
  
  class User(Document):
      name = StringField()
      meta = {'db_alias': 'user-db'}
  
  with switch_db(User, 'archive-user-db') as User:
      User(name='Ross').save()  #  ===》 这时会将数据保存至 'archive-user-db'
  ```

- 切换文档

  `switch_collection()`上下文管理器允许更改集合，允许快速和方便地跨集合访问：

  ```python
  from mongoengine.context_managers import switch_collection
  
  class Group(Document):
      name = StringField()
  
  Group(name='test').save()  # 保存至默认数据库
  
  with switch_collection(Group, 'group2000') as Group:
      Group(name='hello Group 2000 collection!').save()  # 将数据保存至 group2000 集合
  ```

#### 2 定义文档

##### 2.1 定义文档模型(document)

`MongoEngine`允许为文档定义模式，因为这有助于减少编码错误，并允许在可能存在的字段上定义方法。
为文档定义模式，需创建一个继承自`Document`的类，将字段对象作为类属性添加到文档类：

```python
from mongoengine import *
import datetime

class Page(Document):
    title = StringField(max_length=200, required=True)      # ===》 创建一个String型的字段title，最大长度为200字节且为必填项
    date_modified = DateTimeField(default=datetime.datetime.utcnow)      # ===》 创建一个时间类型的字段(utcnow是世界时间，now是本地计算机时间)
```

##### 2.2 定义动态文档模型(Dynamic Document)

动态文档(Dynamic Document)跟非动态文档(document)的区别是，动态文档可以在模型基础上新增字段，但非动态文档不允许新增。

> 动态文档中自定义字段不能以 `_`开头

```python
from mongoengine import *

class Page(DynamicDocument):
    title = StringField(max_length=200, required=True)

# 创建一个page实例，并新增tags字段
>>> page = Page(title='Using MongoEngine')
>>> page.tags = ['mongodb', 'mongoengine']
>>> page.save()      =====》# 不会报错，可以被保存至数据库

>>> Page.objects(tags='mongoengine').count()      =====》# 统计 tags=‘mongengine’的文档数
>>> 1
```

##### 2.3 字段Fields

字段类型包含：

> 以 `》`开头的为常用类型

```python
》BinaryField    #  二进制字段
》BooleanField     # 布尔型字段
》DateTimeField    # 后六位精确到毫妙的时间类型字段
ComplexDateTimeField   # 后六位精确到微妙的时间类型字段
DecimalField    # 
》DictField    # 字典类型字段
》DynamicField   # 动态类型字段，能够处理不同类型的数据
》EmailField     # 邮件类型字段
》EmbeddedDocumentField   # 嵌入式文档类型
》StringField    # 字符串类型字段
》URLField    # URL类型字段
》SequenceField     # 顺序计数器字段，自增长
》ListField       # 列表类型字段
》ReferenceField    # 引用类型字段
LazyReferenceField
》IntField     # 整数类型字段，存储大小为32字节
LongField      # 长整型字段，存储大小为64字节
EmbeddedDocumentListField
FileField  #  列表类型字段
FloatField   # 浮点数类型字段
GenericEmbeddedDocumentField   
GenericReferenceField
GenericLazyReferenceField
GeoPointField
ImageField
MapField
ObjectIdField
SortedListField  
UUIDField
PointField
LineStringField
PolygonField
MultiPointField
MultiLineStringField
MultiPolygonField
```

- 字段通用参数

  - `db_field`：`MongoDB` 字段名称

  - `required`: 是否必填

  - `default`:  默认值

  - `unique`: 是否唯一

  - `unique_with`:唯一字段列表

  - `primary_key`: 主键

  - `choices`：限制该字段的值(为列表、集合或元组中的一个)

  - `validation`：可用于验证字段的值, callable将值作为参数，如果验证失败，则应引发ValidationError

    ```python
    def _not_empty(val):
        if not val:
            raise ValidationError('value can not be empty')
    
    class Person(Document):
        name = StringField(validation=_not_empty)
    ```

- 列表字段(ListField)

  使用ListField字段类型可以向 Document添加项目列表。ListField将另一个字段对象作为其第一个参数，该参数指定可以在列表中存储哪些类型元素：

  ```python
  class  Page(Document)：
      tags = ListField(StringField(max_length = 50))   # ===》 ListField中存放字符串字段
  # 应该可以存放任意类型字段
  ```

- 内嵌文档(Embedded Document)

  MongoDB能够将文档嵌入到其他文档中。要创建嵌入式文档模型，需要继承EmbeddedDocument：

  ```python
  # 创建嵌入式文档模型
  class Comment(EmbeddedDocument):
      content = StringField()
  
  # 创建文档模型，且将 Comment 嵌入Post.comments列表字段中
  class Page(Document):
      comments = ListField(EmbeddedDocumentField(Comment))
  
  comment1 = Comment(content='Good work!')
  comment2 = Comment(content='Nice article!')
  page = Page(comments=[comment1, comment2])
  ```

- 字典字段(Dictionary Fields)

  一般建议使用嵌套文档，这样可以验证数据等等。但是，当你不知道想要存储什么结构时，可以使用字典字段（字典字段不支持验证），字典可以存储复杂数据，其他字典，列表，对其他对象的引用，因此是最灵活的字段类型：

  ```python
  class SurveyResponse(Document):
      date = DateTimeField()
      user = ReferenceField(User)
      answers = DictField()
  
  survey_response = SurveyResponse(date=datetime.utcnow(), user=request.user)
  response_form = ResponseForm(request.POST)
  survey_response.answers = response_form.cleaned_data()
  survey_response.save()
  ```

> 更过字段类型参见：[2.3. Defining documents — MongoEngine 0.24.1 documentation](http://docs.mongoengine.org/guide/defining-documents.html#field-arguments)

##### 2.4 索引 Indexes

为了能在数据库中更快查找数据，需要创建索引。索引可以在模型文档中的meta中指定索引字段以及索引方向。
可以通过在字段名前加上$来指定文本索引。可以通过在字段名前加上＃来指定散列索引

> [2.3. Defining documents — MongoEngine 0.24.1 documentation](http://docs.mongoengine.org/guide/defining-documents.html#global-index-default-options)

```python
class Page(Document):
    category = IntField()
    title = StringField()
    rating = StringField()
    created = DateTimeField()
    meta = {
        'indexes': [
            'title',
            '$title',  #   ====》  文本索引
            '#title',  #     =====》  哈希索引
            ('title', '-rating'),
            ('category', '_cls'),
            {
                'fields': ['created'], 
                'expireAfterSeconds': 3600       #   ====》  设置文档在3600秒后过期，数据库会在3600秒后删除过期文档
            }
        ]
    }
```

- 全局索引默认选项

  ```python
  class Page(Document):
      title = StringField()
      rating = StringField()
      meta = {
          'index_opts': {},
          'index_background': True,
          'index_cls': False,
          'auto_create_index': True,
          'index_drop_dups': True,
      }
  ```

  参数说明：

  - `index_opts`: 设置默认索引选项
  - `index_background`: 为 `True` 时，后台创建索引
  - `index_cls`: 一种关闭 `_cls` 的特定索引的方法
  - `auto_create_index`: 默认为 `True`。`MongoEngine`将确保每次运行命令时`MongoDB`中都存在正确的索引。可以在单独管理索引的系统中禁用此功能。禁用此功能可以提高性能。

##### 2.5 抽象类

如果你想为文档添加一些属性，且不想增加继承。可以使用在 `meta` 中使用 `abstract: True`.

示例：

```python
class BaseDynamicDocument(DynamicDocument):
    """所有model都继承此类, 统一的方法写在此类里"""
    meta = {'abstract': True}

    @classmethod
    def get_by_id(cls, _id, only_active=True, raise_error=False):
        _obj = None
        _obj = cls.find_one(only_active=only_active, _id=_id)
        if not _obj and raise_error:
            table_name = getattr(cls, 'TABLE_NAME', None)
            if not table_name:
                raise Exception('%s类没有指定TABLE_NAME' % cls.__name__)
            content = 'get_by_id not found object. class_name: %s, table_name: %s, _id: %s' % \
                      (cls.__name__, table_name, _id)
            logger_error.error(content)
            raise BaseError(RetCodeAndMessage.Common.miss_obj(table_name))
        return _obj

    @classmethod
    def find_one(cls, only_active=True, **kwargs):
        return cls.find(only_active=only_active, **kwargs).limit(1).first()

    @classmethod
    def find(cls, only_active=True, **kwargs):
        if only_active and 'deleted' not in kwargs and getattr(cls, 'deleted', None) is not None:
            # 踢出被逻辑删除的文档
            if isinstance(getattr(cls, 'deleted', None), BooleanField):
                kwargs.update({'deleted__ne': True})
            elif isinstance(getattr(cls, 'deleted', None), IntField):
                kwargs.update({'deleted__ne': 1})
        return cls.objects(**kwargs)

    @classmethod
    def create(cls, **kwargs):
        if 'ts' not in kwargs.keys():
            ts = time.time()
        else:
            ts = kwargs.get('ts')
        if isinstance(getattr(cls, 'ts', None), FloatField) and 'ts' not in kwargs.keys():
            kwargs['ts'] = ts
        if isinstance(getattr(cls, 'create_time', None), StringField) and 'create_time' not in kwargs.keys():
            kwargs['create_time'] = DateUtil.timestamp_to_datetime_str(ts)
        if isinstance(getattr(cls, 'dt', None), StringField) and 'dt' not in kwargs.keys():
            kwargs['dt'] = DateUtil.timestamp_to_datetime_str(ts, fmt=DateUtil.DATE_FORMAT)
        if isinstance(getattr(cls, 'create_dt', None), StringField) and 'create_dt' not in kwargs.keys():
            kwargs['create_dt'] = DateUtil.timestamp_to_datetime_str(ts, fmt=DateUtil.DATE_FORMAT)
        return cls(**kwargs)

    @classmethod
    def get_or_create(cls, _id, only_active=True):
        obj = cls.get_by_id(_id, only_active=only_active)
        if not obj:
            pk_name = getattr(cls, '_meta', {}).get('id_field')
            obj = cls.create(**{pk_name: _id})
        return obj

    def delete(self, db_del=False, **write_concern):
        if not db_del and getattr(self, 'deleted', None) is not None:
            if hasattr(self, 'delete_time'):
                setattr(self, 'delete_time', DateUtil.datetime_to_str())
            if isinstance(getattr(self, 'deleted', None), BooleanField) \
                    or isinstance(getattr(self, 'deleted', None), bool):
                setattr(self, 'deleted', True)
                super(BaseDynamicDocument, self).save()
            elif isinstance(getattr(self, 'deleted', None), IntField) \
                    or isinstance(getattr(self, 'deleted', None), int):
                setattr(self, 'deleted', 1)
                super(BaseDynamicDocument, self).save()
            else:
                DingTalkMessage().send_msg(
                    'class:%s deleted type:%s' % (self.__name__, type(getattr(self, 'deleted', None))))
        else:
            super(BaseDynamicDocument, self).delete(**write_concern)

    @classmethod
    def delete_batch(cls, db_del=False, **kwargs):
        """批量删除"""
        if not db_del and getattr(cls, 'deleted', None) is not None:
            update_str = {}
            now = DateUtil.datetime_to_str()
            if hasattr(cls, 'delete_time'):
                update_str['delete_time'] = now
            if isinstance(getattr(cls, 'deleted', None), BooleanField):
                update_str['deleted'] = True
            elif isinstance(getattr(cls, 'deleted', None), IntField):
                update_str['deleted'] = 1
            else:
                DingTalkMessage().send_msg(
                    'class:%s deleted type:%s' % (cls.__name__, type(getattr(cls, 'deleted', None))))
            cls.find(**kwargs).update(**update_str)
        else:
            cls.find(**kwargs).delete()

    @classmethod
    def get_batch_by_id(cls, ids, only_active=True):
        """批量获取obj_dict"""
        if not ids:
            return {}
        objs = cls.find(_id__in=ids, only_active=only_active)
        objs_dict = {obj.id: obj for obj in objs}
        return objs_dict

    @queryset_manager
    def objects_not_delete(doc_cls, queryset):
        return queryset.filter(deleted=0)
```

##### 2.6 文档实例

实例化一个对象。

```python
class Page(Document):
    title = StringField(max_length=200, required=True)

    meta = {'allow_inheritance': True}
    
>>> page = Page(title="Test Page")
>>> page.title
'Test Page'
```

- 持久化和删除文档

  使用`save()`方法即可持久化数据

  ```python
  >>> page = Page(title="Test Page")
  >>> page.save()  # 保存
  >>> page.title = "My Page"
  >>> page.save()  # 修改title值后 再次保存
  >>> page.id
  ObjectId('123456789abcdef000000000')
  >>> page.pk  # 查找主键
  ObjectId('123456789abcdef000000000')
  ```

  使用 `delete` 即可删除文档

  ```python
  delete(signal_kwargs=None, **write_concern)
  
  # 示例
  p = Page.objects(title='My Page').first()
  p.delete()
  ```

##### 2.7 查询文档

`Document`类具有一个`objects`属性，用于访问与类关联的数据库中的对象。该`objects`属性实际上是一个 `QuerySetManager`，它在访问时创建并返回一个新的`QuerySet`对象, 可以迭代该对象以从数据库中获取文档

使用示例：

```python
# 查询出国家是英国的所有用户
uk_users = User.objects(country='uk')

# auther是一个嵌入文档字段，country是嵌入文档的field，通过auther__country的方式访问值
uk_pages = Page.objects(author__country='uk')

# 查询18岁以下的用户
young_users = Users.objects(age__lte=18)
```

常用的运算符如下：

- `ne`: 不等于
- `lt`：小于
- `lte`： 小于等于
- `gt`: 大于
- `gte`： 大于等于
- `in`:  在列表中
- `nin`：不再值列表中
- `all`： 提供的值列表中的每个项目都在数组中（查询结果集为给定值集的子集）
- `size`： 数组大小
-  `exists`: 字段值存在

###### **字符串查询** 

以下运算符可用作使用正则表达式查询：

- `exact` - 字符串字段与值完全匹配
- `iexact` - 字符串字段与值完全匹配（不区分大小写）
- `contains` - 字符串字段包含值
- `icontains` - 字符串字段包含值（不区分大小写）
- `startswith` - 字符串字段以值开头
- `istartswith` - 字符串字段以值开头（不区分大小写）
- `endswith` - 字符串字段以值结尾
- `iendswith` - 字符串字段以值结尾（不区分大小写）
- `match` - 执行$ elemMatch，以便您可以匹配数组中的整个文档

###### 原始查询

如果希望使用pymongo操作数据库，可以使用__raw__关键字：

```python
Page.objects(__raw__={'tags': 'coding'})
```

> [PyMongo — MongoDB Drivers](https://www.mongodb.com/docs/drivers/pymongo/)

###### 限制与跳过查询

```python
# 查询前五条文档
users = User.objects[:5]

# 查询第六条以后的所有文档
users = User.objects[5:]

# 查询第十一到第十五条文档
users = User.objects[10:15]

>>> # 确认数据库中不存在文档
>>> User.drop_collection()
>>> User.objects[0]
IndexError: list index out of range   ===》 报IndexError的错
>>> User.objects.first() == None
True
>>> User(name='Test User').save()
>>> User.objects[0] == User.objects.first()
True
```

###### 聚合查询

常用聚合查询示例：

```python
# 求总数
num_users = User.objects.count()

# 求和
yearly_expense = Employee.objects.sum('salary')

# 求平均值
mean_age = User.objects.average('age')
```



###### 限制查询

搜索文档子集

```python
>>> class Film(Document):
...     title = StringField()
...     year = IntField()
...     rating = IntField(default=3)
...
>>> Film(title='The Shawshank Redemption', year=1994, rating=5).save()
>>> f = Film.objects.only('title').first()
>>> f.title
'The Shawshank Redemption'
>>> f.year   # None
>>> f.rating # default value
3
```



```python
post = BlogPost.objects.exclude('title').exclude('author.name')
# 查询出除了title和auther.name字段的内容
```



###### 高级查询

如果希望通过 or 或者 and 来多条件查询时，需要使用 Q(条件语句1) | Q(条件语句2) Q(条件语句1) | Q(条件语句2)

```python
from mongoengine.queryset.visitor import Q

# 获取已发布的文档
Post.objects(Q(published=True) | Q(publish_date__lte=datetime.now()))

# 获取 featured为真 同时 hits大于等于1000或大于等于5000  的文档
Post.objects((Q(featured=True) & Q(hits__gte=1000)) | Q(hits__gte=5000))
```



###### 原子更新

更新方法有：`update(), update_one(), modify()`
更新修饰符有：

- `set` - 重新设置一个值
- `unset` - 删除
- `inc` - 加
- `dec` - 减
- `push` - 将新值添加到列表中
- `push_all` - 将多个值添加到列表中
- `pop` - 根据值删除列表的第一个或最后一个元素
- `pull` - 从列表中删除值
- `pull_all` - 从列表中删除多个值
- `add_to_set` - 仅当列表中的值不在列表中时才为其添加值

```python
>>> post = BlogPost(title='Test', page_views=0, tags=['database'])
>>> post.save()
>>> BlogPost.objects(id=post.id).update_one(inc__page_views=1)
>>> post.reload()  # 值已被修改，重新加载数据
>>> post.page_views
1
>>> BlogPost.objects(id=post.id).update_one(set__title='Example Post')
>>> post.reload()
>>> post.title
'Example Post'
>>> BlogPost.objects(id=post.id).update_one(push__tags='nosql')
>>> post.reload()
>>> post.tags
['database', 'nosql']
```

⚠️注意：如果未指定修饰运算符，则默认为 `$set`.即下面两种写法是一样的

```python
>>> BlogPost.objects(id=post.id).update(title='Example Post')
>>> BlogPost.objects(id=post.id).update(set__title='Example Post')
```

> [MongoEngine中文文档 - zhenyuantg - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhenyauntg/p/13201826.html)

### `celery`

[Celery - 简书 (jianshu.com)](https://www.jianshu.com/p/620052aadbff)



## 抓包开发

因为该项目客户端为`Android App`，所以`App`访问的资源需要通过抓包工具进行确认。

目前公司内所使用的工具是`mac OS` 上的`Charles`,使用该工具只需将手机上的 `wifi` 配置好指定的代理即可。

> 可参见此文档进行设置：[Charles 手机抓包记录](https://juejin.cn/post/6844904106255974413)

配置完成后，打开`App`即可看到抓包记录。


