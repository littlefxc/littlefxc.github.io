---
title: Spring-Security-Oauth2令牌增加额外信息
date: 2019-06-26 11:13:38
categories: oauth2
tags:
- oauth2
- spring-boot
---

在实现了 Oauth2 后，我想要在令牌增加中额外信息，那么该怎么做？

下面是我的做法，首先实现 `org.springframework.security.oauth2.provider.token.TokenEnhancer` 接口：

```java
package com.fengxuechao.examples.auth.config;

import org.springframework.security.oauth2.common.DefaultOAuth2AccessToken;
import org.springframework.security.oauth2.common.OAuth2AccessToken;
import org.springframework.security.oauth2.provider.OAuth2Authentication;
import org.springframework.security.oauth2.provider.token.TokenEnhancer;

import java.util.HashMap;
import java.util.Map;

import static org.apache.commons.lang3.RandomStringUtils.randomAlphabetic;

/**
 * token 额外信息
 *
 * @author fengxuechao
 * @version 0.1
 * @date 2019/5/16
 */
public class CustomTokenEnhancer implements TokenEnhancer {
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        Map<String, Object> additionalInfo = new HashMap<String, Object>();
        additionalInfo.put("organization", authentication.getName() + randomAlphabetic(4));
        ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(additionalInfo);
        return accessToken;
    }
}
```

然后在 `AuthorizationServerConfigurerAdapter` 认证服务代码中配置：

```java
public class AuthorizationServerConfigInJwt extends AuthorizationServerConfigurerAdapter {
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // token 携带额外信息
        TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
        tokenEnhancerChain.setTokenEnhancers(Arrays.asList(tokenEnhancer(), jwtTokenEnhancer()));
        endpoints
                .tokenStore(tokenStore())
                .tokenEnhancer(tokenEnhancerChain)
                .userDetailsService(userDetailsService)
                .authenticationManager(authenticationManager)
                .setClientDetailsService(clientDetailsService);
    }
    
    /**
     * Token 额外信息
     *
     * @return
     */
    @Bean
    public TokenEnhancer tokenEnhancer() {
        return new CustomTokenEnhancer();
    }
    
    /**
     * jwt token：使用了非对称密钥对来签署令牌:
     * 1.生成 JKS Java KeyStore 文件：keytool -genkeypair -alias jwt_rsa -keyalg RSA -keypass 123456 -keystore jwt_rsa.jks -storepass 123456
     * 2.导出公钥：keytool -list -rfc --keystore jwt_rsa.jks | openssl x509 -inform pem -pubkey
     * 3.将 PUBLIC KEY 保存至 public.txt
     *
     * @return
     */
    @Bean
    public JwtAccessTokenConverter jwtTokenEnhancer() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setKeyPair(new KeyStoreKeyFactory(resource, keyStorePass.toCharArray()).getKeyPair(keyPairAlias));
        // 使用对称密钥来签署令牌
        // converter.setSigningKey("fengxuechao.littlefxc");
        return converter;
    }
}
```

或者 `tokenServices.setTokenEnhancer(tokenEnhancer);`

最后演示一下最终效果：

```json
{
  "access_token": "4aae3856-bc33-4e4d-86bc-eb475fc45569",
  "token_type": "bearer",
  "refresh_token": "fe2ed35d-5c53-4610-abb7-c1053cba6803",
  "expires_in": 119,
  "scope": "read",
  "organization": "userAKqz"
}
```

jwt 

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsicmVhZCJdLCJvcmdhbml6YXRpb24iOiJ1c2VyZWx2ayIsImV4cCI6MTU2MDQ4NDI0OCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjliNTU2ZTBiLTZlZmQtNDkwZC05OGMwLWIzYzYwNjM2ZDczMCIsImNsaWVudF9pZCI6ImNsaWVudCJ9.oaqlviXcQPCLAZP8cV7v-WA75AoiodiG6d2WR9yqJhOFCg7LDsnCjk63J59sq434CZHRIOkCgMi2hVJHOc4MTIFce61Kk046G3-yK313CtMy5LWeVXdKbAHH0gcuoDO3OCJ7u7GzngPtA6bVfxjJFNJ6MmFxEnFPjB5dos9Bb8zYduE2ELMH2aTCS-67R_aQ0BCZaYo5NMH1_jqz9d1hI_kpBx3auR_d2Vh1eJiC_f9Z-rTmRvXdwQefhwgXZ1UCWjV0NuoCqFO3KicEhjGOkqXZ5eh0vGR5zKwKJfCys1lNgXjXVVntHYkXt96ymQ9477pCAWCONZsbkM7244500Q",
  "token_type": "bearer",
  "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsicmVhZCJdLCJvcmdhbml6YXRpb24iOiJ1c2VyZWx2ayIsImF0aSI6IjliNTU2ZTBiLTZlZmQtNDkwZC05OGMwLWIzYzYwNjM2ZDczMCIsImV4cCI6MTU2MDQ4NzcyOCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjJhNTUxOWJjLWUzYTAtNGJjOC1hNTRkLTlmMDNiMjYwNjZkNCIsImNsaWVudF9pZCI6ImNsaWVudCJ9.BgY6N0kzxVApFD-C7UVMDmczSoMY9tglnKzTkybfneoeAAs8ljftIwA5sPWua28Xhl-MNAQ9HL6Q6ou-EbgFlcHC2uPPbJ5silnPLPdTnvVko9l-8w-3WLPk96YbODdQemqFZHSrR1lPmXHB5sR7QjncxGxvuSYZEtPXxZz39lJbyQLSflXADqlk4ZV3BxS-M7d8FcTJEM1uTgwUBSns2N6AZnTkd2FnGskadaV2qhky5TznJjQqRETVS8xCiZCFYwCq5sAMHOj-_BrwlmCeoPfcy38ofbql-qVWfQJiAeU7yWLlAu_hd-zRIIbv-dqRmSF9T9rCxVPv84ptddO1Hw",
  "expires_in": 119,
  "scope": "read",
  "organization": "userelvk",
  "jti": "9b556e0b-6efd-490d-98c0-b3c60636d730"
}
```

最终返回的 Token 信息中多了一个属性 `organization`，结果符合期望结果。