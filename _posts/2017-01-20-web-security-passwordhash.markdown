#**shiro 密码保存以及比较流程**

在现实例子中，由于数据库密码泄漏而导致注册用户密码泄漏的事件经常发生。为了防止明文直接保存在数据表中，用户输入的密码，都需要Hash之后，保存在表中。同时，为了保护Hash之后的密码被暴力破解，同时还需要加点salt。
下面会以webside中的处理流程为例子，简要说明明文的密码，如何变成保存在表中的Hash字段

###1. 密码保存


Hash算法关键信息(md5Password)
- 明文的密码
- 随机数作为Salt(注意，在Md5Hash算法中，在Salt上又加了一层Salt--UserName。)
- Hash迭代次数

	Hash函数的返回值是byte[]，为了能够以字符串方式表达，使用Base64编码。
	Base64 encodes the specified byte array and then encodes it as a String using Shiro's preferred character encoding (UTF-8).
	
通过上面的描述，我们可以清楚，保存在数据库表中的密码经历了如下处理

	MD5( plain_password, salt[ userName + Random.toHex()],hashIterations).toBase64



###2. 密码比较

在用户需要登录时，需要将用户传递过来的plaintext按照上述的规则，转换为Base64的字符串。在当前代码中，由于套用了shiro的Authentication流程，看起来有些曲折。摘取流程如下。

核心流程在HashedCredentialsMatcher::doCredentialsMatch

	    public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
		Object tokenHashedCredentials = hashProvidedCredentials(token, info);
		Object accountCredentials = getCredentials(info);
		return equals(tokenHashedCredentials, accountCredentials);
	    }
	    
- AuthenticationInfo来源于MyRealm::doGetAuthenticationInfo(token)方法，需要注意的时，在该函数中构造AuthenticationInfo时，其Salt的来源：
 
             authenticationInfo = new SimpleAuthenticationInfo(
            		userEntity, // 用户对象
            		userEntity.getPassword(), // 密码
					ByteSource.Util.bytes(username + userEntity.getCredentialsSalt()),// salt=username+salt
					getName() // realm name
			);
			
是用户表中的salt字段，前面加上用户名称。

- AuthenticationToken来源于登录信息，里面包含用户名及密码.

#### 处理流程
- hashProvidedCredentials(token,info)  该函数是根据Hash参数（hash算法，密码，salt,  迭代测试），构造SimpleHash对象。
- getCredentials(info)  该函数根据AuthenticationInfo中Hash之后的byte[],  利用Base64.decode解码；再反构出SimpleHash对象。

		   protected Object getCredentials(AuthenticationInfo info) {
			Object credentials = info.getCredentials();

			byte[] storedBytes = toBytes(credentials);

			if (credentials instanceof String || credentials instanceof char[]) {
			    //account.credentials were a char[] or String, so
			    //we need to do text decoding first:
			    if (isStoredCredentialsHexEncoded()) {
				storedBytes = Hex.decode(storedBytes);
			    } else {
				storedBytes = Base64.decode(storedBytes);
			    }
			}
			AbstractHash hash = newHashInstance();
			hash.setBytes(storedBytes);
			return hash;
		    }


	    
	    


