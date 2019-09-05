#### 安装
```
composer require laravel/passport=~4.0
```
 notes:   
  1）确保系统安装unzip、zip等命令。  
  2）composer 安装出现 Authentication required (packagist.phpcomposer.com) 问题，修改composer.json 中的源，repositories.packagist.url = https://packagist.laravel-china.org 。  

#### 注册服务提供者
在config/app.php的providers 数组中加入 Laravel\Passport\PassportServiceProvider::class  

#### 迁移数据库
```
php artisan migrate  //生成用于存储客户端和令牌的数据表
```
#### 生成加密健
```
 php artisan passport:install  
```
1、生成oauth-private.key（用于构建认证服务器），oauth-public.key（用于构建资源服务器）  
2、oauth_clients数据库生成「个人访问」客户端和「密码授权]两条数据。  

#### 配置Passport（参考官方文档）
在Model中，我们需要增加 HasApiTokens class  
在AuthServiceProvider中， 增加 "Passport::routes()"  
在 auth.php中， 更改 api 认证方式为passport  

#### 申请客户端以及私人访问令牌 （两种方式）
###### 1. 命令形式（不方便客户注册）
```
php artisan passport:client
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Laravel/8.jpg)  

###### 2. Passport Vue 组件 
```
php artisan vendor:publish --tag=passport-components  //发布 Passport Vue，组件位于resources/assets/js/components下
```
//注册到resources/assets/js/app.js 文件，记得要放在new Vue上面  
```
Vue.component(
    'passport-clients',
    require('./components/passport/Clients.vue')
);

Vue.component(
    'passport-authorized-clients',
    require('./components/passport/AuthorizedClients.vue')
);

Vue.component(
    'passport-personal-access-tokens',
    require('./components/passport/PersonalAccessTokens.vue')
);
```
 //编译前端资源
```
npm install   //此处报错，移步larravel Mix文档
npm run dev
```
编译后资源放在public/js/app.js下  

//组件放入应用模板（记得引入编译后的app.js）  
```
<passport-clients></passport-clients>
<passport-authorized-clients></passport-authorized-clients>
<passport-personal-access-tokens></passport-personal-access-tokens>
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Laravel/9.jpg)  
以上认证服务器都已经搭建完成  

#### 第三方应用实现登录
###### 1. 申请客户端  
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Laravel/10.jpg)  
回调地址 http://third.plat.goods/dew/sso  

###### 申请授权码和访问令牌
//获取授权码 code （第一次交互）  
```
$query = http_build_query(array(
        'client_id' => 3,
        'redirect_uri' => 'http://third.plat.goods/dew/sso', //地址必须为上面的回调地址
        'response_type' => 'code',  //固定值
        'scope' => '',
        'state' => urlencode('http://laravel.plat.goods/user')   //可以放用户访问的地址。
));

return redirect('http://laravel.plat.goods/oauth/authorize?'.$query);  ///laravel.plat.goods为上面认证服务器
```
![image](https://raw.githubusercontent.com/SexyPhoenix/Blog/master/static/Laravel/11.jpg)  
//获取访问令牌  access token 以及向资源服务器请求用户信息   
授权后会重定向回调地址  
```
Route::get('/dew/sso', 'SSOController@callback');  //路由文件里添加
php artisan make:controller SSOController //创建文件
```
```
<?php
namespace App\Http\Controllers;

use App\Models\User;
use GuzzleHttp\Client;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class SSOController extends Controller
{
    protected $http;
    public function __construct()
    {
        $this->http = new Client();
    }

    /**
     * 获取授权码后的回调URL
     * @param Request $request
     * @return \Illuminate\Http\RedirectResponse
     */
    public function callback(Request $request)
    {

        $token = $this->token($request); //第二次交互
        $login = $this->login($token);//第三次交互

        if($login){

            if($request_url = $request->input('state', null)){
                $request->session()->put('url.intended', urldecode($request_url));
            }
            return redirect()->intended();  //跳转到 http://laravel.plat.goods/user
        }else{
            return redirect()->to('http://laravel.plat.com/home/public/login'); //服务提供商网站必须登录
        }
    }

    /**
     * 获取access token
     * @param $request
     * @return array|mixed
     */
    protected function token($request)
    {
        $code = $request->code;
        if($code) {

            try {

                $response = $this->http->post('http://laravel.plat.goods/oauth/token', [
                    'form_params' => [
                        'grant_type' => 'authorization_code',  //固定值
                        'client_id' => 3,
                        'client_secret' => 'UihXNHoSqohdtQ8Js6Av7AOyk3GBNB9rJziDPaWf',
                        'redirect_uri' => 'http://third.plat.goods/dew/sso',
                        'code' => $code,
                    ],
                ]);

                $response_data = json_decode((string)$response->getBody(), true);

                return $response_data;
            } catch (\Exception $e) {

                Log::error('get token by code failed: '.$code.' - '.$e->getMessage().' - '.$e->getTraceAsString());

                return [];
            }
        }else{

            return [];
        }

    }
    /**
     * 通过token获取用户信息，并进行登录操作
     * @param $token
     * @return bool
     */
    protected function login($token)
    {
        if(empty($token)) return false;

        $access_token = $token['access_token'];
        try {
           
           // 资源服务器和认证服务器放在了一起，可以独立。
            $response = $this->http->request('GET', 'http://laravel.plat.goods/api/user', [
                'headers' => [
                    'Accept' => 'application/json',
                    'Authorization' => $token['token_type'] . ' ' . $access_token,
                ]
            ]);
            $users_body = $response->getBody();
            $data = json_decode($users_body, true);
            if($data) {
                $user = new User($data);

                //because of employee_id is guarded
                $user->setAttribute($user->getKeyName(), $data['employee_id']);
                //login user in my system
                auth()->login($user, false);

                return true;
            }else{

                return false;
            }
        }catch (\Exception $e){

            Log::error('get user failed by access_token:'.$access_token.'|'.$e->getMessage());
            return false;
        }
    }
}
```

 //设置资源文件
```
Route::middleware('auth:api')->get('/user', 'UserController@user'); //routes/api.php文件中设置
php artisan make:controller UserController //创建文件
```
```
class UserController extends Controller
{
    public function user(Request $request)
    {
        return $request->user();
    }
}
```






