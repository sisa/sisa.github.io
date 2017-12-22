---
layout: post
title: JWT Token Spring Boot Api üzerinde kimlik doğrulama!
tags: [spring, jwt, security, rest, authentication, authorization, kimlik dogrulama]
bigimg: 

---

Microservices uygulama geliştirmede yaygın olarak kullanılan **JWT(JSON Web Token)** ile **Spring Boot** kullanarak yapmış olduğum kimlik doğrulama ve Rest servis yetkinlendirme örneğini anlatacağım. Uygulamanın kaynak kodları paylaşıldığı için 
domain objeleri(entities), repo ve servis  gb. sınıfları adım adım oluşturulmasını anlatmayacağım.

travis | codecov | git
------ | ------- | ---
[![Build Status](https://travis-ci.org/sisa/spring-security-with-jwt.svg?branch=master)](https://travis-ci.org/sisa) | [![Codecov branch](https://codecov.io/gh/sisa/spring-security-with-jwt/branch/master/graphs/badge.svg)](https://codecov.io/gh/sisa/spring-security-with-jwt) | [![Repo](https://sisa.github.io//img/GitHub-Mark-32px.png)](https://github.com/sisa/spring-security-with-jwt)

## Gereksinimler    

   + Maven 3 
   + JDK 1.8    
   
## API Kullanımına Genel Bakış

```curl
# admin kullanıcı için jwt token isteği
# password JWT ile encode edilmiştir.
curl -X POST \
  http://localhost:5300/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{
"username":"admin",
"password":"$2a$10$HG0tQJHWZZen.kerZYz1rePqDx8EjI7LO.pDJOjF3udpWPTbfODF2"
}'

# admin için aldığımız jwt token ile api/cities/1 get isteğini atıyoruz.
# eğer yetkisi varsa json dönecetir.
curl -X GET \
  http://localhost:5300/api/cities/1 \
  -H 'Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJhZG1pbiIsImF1ZCI6IndlYiIsImlzcyI6IlNUU19BdXRoX1NlcnZlciIsImV4cCI6MTUxMzc1NjIzOCwiaWF0IjoxNTEzNzU2MTQ4LCJqdGkiOiI3NGM4MjRlMC02ZTgzLTQ4MWItOGEzMS1hYTA1YmRlMjdmNjQifQ.XXqB7E0tHyJJOg2fErsjUSZcq3W2Z-VmgJwgBR7b4pFdwMyU8DOl2x-ettn53YAFAAS1D5gPCPuPossTZxY3jQ' \
  -H 'Content-Type: application/json' \
```

Benzer şekilde admin yerine ROLE_STANDART rolündeki kullanıcı için önce login olup token elde edebilirsiniz. Sonrada şehir sorgulama apisi curl yada postman üzerinden test edersiniz.

İki farklı api türümüz var. **io.sisa.api.controller** paketi altında :
+ UserController
+ CityController

Bu apiler ark tarafta UserDetailsService ve CityService servislerini kullanıyor.

Uygulama dışa bağımlılığı olmaması adına embeded db kullanıldı(h2 database engine).
Uygulama ayapa kalkarken ilgili db scriptleri otamatik çalıştırılıyor.
İlgili ayarlar **io.sisa.core.datasource.DataSourceConfig** sınıfında mevcut.

## WebSecurityConfigurerAdapter  

UYgulama içinde WebSecurityConfigurerAdapter'dan extend eden WebSecurity sınıfında override ettiğimiz **configure**

methodu :

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
	http.csrf().disable()
			.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
			.and()
			.exceptionHandling().authenticationEntryPoint(restAuthenticationEntryPoint)
			.and()
			.authorizeRequests()
			.antMatchers(HttpMethod.POST, "/auth/login").permitAll()
			.antMatchers("/cities/*").hasRole("ADMIN")
			.antMatchers("/cities").hasRole("STANDARD")
			.anyRequest().authenticated()
			.and()
			.addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class);

}
```
Bu metodda hangi apilere erişim verip vermiyeceğimiz belirliyoruz. Ayrıca token validate edibilmek için Filter ekliyoruz.
Bunlar sırasıyla :
+ ``` antMatchers(HttpMethod.POST, "/auth/login").permitAll()``` :
Login olup token alabilmek için /api/auth/login  api için("/auth/login") herhangi security kısıtı olmadan erişimine izin veriyoruz. 

+ ``` .antMatchers("/cities/*").hasRole("ADMIN").antMatchers("/cities").hasRole("STANDARD") ``` :
/api/cities ve /api/cities/{cityId}  apilerine istek gönderebilmek için ADMIN ve STANDART role kullanıcılarına sahip olması gerekiyor.

+ ``` .addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class) ``` : 

Bu filter sınıfı içinde gelen isteklerin headerinda token kontrölü yapılıyor. **AuthenticationTokenFilter** sınıfında doFilterInternal metodunu override edip gerekli kontrölleri yazıyoruz.

## AuthenticationTokenFilter

```java
@Override
protected void doFilterInternal(HttpServletRequest request,HttpServletResponse response,FilterChain chain) throws IOException, ServletException {

	final String requestHeader = request.getHeader(jwtTokenHelper.getKeyFromProperties(JwtProperties::getHeader));

	String username = null;
	String authToken = null;
	if (requestHeader != null && requestHeader.startsWith("Bearer ")) {
		authToken = requestHeader.substring(7);
		try {
			username = jwtTokenHelper.getUsernameFromToken(authToken);
		} catch (IllegalArgumentException e) {
			logger.error("an error occured during getting username from token", e);
		} catch (ExpiredJwtException e) {
			logger.warn("the token is expired", e);
		}
	} else {
		logger.warn("couldn't find bearer string, will ignore the header");
	}

	if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {

		UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
		if (jwtTokenHelper.validateToken(authToken, userDetails)) {
			UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(userDetails,
					null,
					userDetails.getAuthorities());

			auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
			SecurityContextHolder.getContext().setAuthentication(auth);
		}
	}

	chain.doFilter(request, response);

}
```

Birinde bölümde headerda token parse ediliyor.Token varsa ikinci bölümde user db'den kontrol ediliyor ve jwtTokenHelper sınıfı yardımıyla token valide ediliyor. Eğer başarılı olursa spring security context doğrulandığı setleniyor. Böylelikle cities api'lerine yapılan istekler doğrulanmış olarak çağrılabiliyor.

## JwtTokenHelper

Son olarak jwt token işlemleri için yazdığım helper sıfında ki bir iki metodu inceleyelim.

```java
 public String generateToken(UserDetails userDetails, Device device) {
        Map<String, Object> claims = new HashMap<>();
        return doGenerateToken(claims, userDetails.getUsername(), generateAudience(device));
    }

    private String doGenerateToken(Map<String, Object> claims, String subject, String audience) {
        final Date createdDate = new Date();
        final Date expirationDate = calculateExpirationDate(createdDate);
        

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setAudience(audience)
                .setIssuedAt(createdDate)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS512, jwtProperties.getSecret())
                .setId(UUID.randomUUID().toString())
                .setIssuer(jwtProperties.getAuthServerName())
                .compact();
    }
    
   public String getUsernameFromToken(String token) {
        return getClaimFromToken(token, Claims::getSubject);
   }
    
   public <T> T getClaimFromToken(String token, Function<Claims, T> claimsResolver) {
	final Claims claims = getAllClaimsFromToken(token);
	return claimsResolver.apply(claims);
   }
    
   public Boolean validateToken(String token, UserDetails userDetails) {

        final String username = getUsernameFromToken(token);

        return (
                username.equals(userDetails.getUsername())
                        && !isTokenExpired(token)
        );
    }
```

+ jwt token oluşturma ```doGenerateToken ``` : subject kısmına sadede username koyuyoruz. secret şifresi ve issuer parametreleri için yml dosyasından uygulama açışında jwtProperties sınıfına maplediğimiz değerleri kullanıyoruz.
+ jwt validasyonu için ``` validateToken ``` 




