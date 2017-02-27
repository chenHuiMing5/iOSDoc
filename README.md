# iOSDoc
ios 开发过程中遇到的问题
总结


#iOS 对https的总结
苹果规定2017年1月1日以后，所有APP都要使用HTTPS进行网络请求，否则无法上架，因此研究了一下在iOS中使用HTTPS请求的实现。相信大家对HTTPS都或多或少有些了解，这里我就不再介绍了，主要功能就是将传输的报文进行加密，提高安全性。
##1.证书准备
证书分为两种，一种是花钱向认证的机构购买的证书，服务端如果使用的是这类证书的话，那一般客户端不需要做什么，用HTTPS进行请求就行了，苹果内置了那些受信任的根证书的。另一种是自己制作的证书，使用这类证书的话是不受信任的（当然也不用花钱买），因此需要我们在代码中将该证书设置为信任证书。

我这边使用的是xca来制作了根证书，制作流程请参考http://www.2cto.com/Article/201411/347512.html，由于xca无法导出.jsk的后缀，因此我们只要制作完根证书后以.p12的格式导出就行了，之后的证书制作由命令行来完成。自制一个批处理文件，添加如下命令：

然后在文件夹空白处按住ctrl+shift点击右键，选择在此处打开命令窗口，在命令窗口中输入“start.bat ip/域名”来执行批处理文件，其中start.bat是添加了上述命令的批处理文件，ip/域名即你服务器的ip或者域名。执行成功后会生成一个.jks文件和一个以你的ip或域名命名的文件夹，文件夹中有一个.cer的证书，这边的.jks文件将在服务端使用.cer文件将在客户端使用，到这里证书的准备工作就完成了。

2、服务端配置
由于我不做服务端好多年，只会使用Tomcat，所以这边只讲下Tomcat的配置方法，使用其他服务器的同学请自行查找设置方法。

打开tomcat/conf目录下的server.xml文件将HTTPS的配置打开，并进行如下配置：


keystoreFile是你.jks文件放置的目录，keystorePass是你制作证书时设置的密码，netZone填写你的ip或域名。注意苹果要求协议要TLSv1.2以上。

3、iOS端配置
首先把前面生成的.cer文件添加到项目中，注意在添加的时候选择要添加的targets。

1.使用NSURLSession进行请求

// /先导入证书
NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"server" ofType:@"cer"];//证书的路径
NSData *certData = [NSData dataWithContentsOfFile:cerPath];

// AFSSLPinningModeCertificate 使用证书验证模式
AFSecurityPolicy *securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeCertificate];

// allowInvalidCertificates 是否允许无效证书（也就是自建的证书），默认为NO
// 如果是需要验证自建证书，需要设置为YES
securityPolicy.allowInvalidCertificates = NO;

//validatesDomainName 是否需要验证域名，默认为YES；
//假如证书的域名与你请求的域名不一致，需把该项设置为NO；如设成NO的话，即服务器使用其他可信任机构颁发的证书，也可以建立连接，这个非常危险，建议打开。
//置为NO，主要用于这种情况：客户端请求的是子域名，而证书上的是另外一个域名。因为SSL证书上的域名是独立的，假如证书上注册的域名是www.google.com，那么mail.google.com是无法验证通过的；当然，有钱可以注册通配符的域名*.google.com，但这个还是比较贵的。
//如置为NO，建议自己添加对应域名的校验逻辑。
securityPolicy.validatesDomainName = YES;

NSSet * certSet = [[NSSet alloc] initWithObjects:certData, nil];

securityPolicy.pinnedCertificates = certSet;

##2 设置单列
+ (instancetype)sharedClient
{
static BikeNetworkAPIClient *_shareClient = nil;
static dispatch_once_t onceToken;//线程安全

dispatch_once(&onceToken, ^{
_shareClient = [[BikeNetworkAPIClient alloc] initWithBaseURL:[NSURL URLWithString:DD_API_SERVER]];
_shareClient.securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeNone];
//        [_shareClient.responseSerializer setAcceptableContentTypes:[NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript",@"text/html", @"text/plain",nil]];
_shareClient.responseSerializer.acceptableContentTypes = [NSSet setWithObjects:@"application/xml", @"application/json", @"text/html",@"text/json",@"text/javascript", @"text/plain", nil];

//加上这行代码，https ssl 验证。
[_shareClient setSecurityPolicy:[self customSecurityPolicy]];

});

return _shareClient;
}
