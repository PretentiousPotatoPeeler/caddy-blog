---
title: "Mutual SSL in Spring Boot"
date: 2019-08-01T11:59:34+02:00
---

This article will contain info on how to enable Mutual SSL in Spring Boot.

# 1. Prerequisites
1. crt (or cer) and key file
2. OpenSSL installed 
3. Keytool installed

# 2. Generate trust and keystore 
We first need to generate a p12 truststore. This will house both the required certificates as well as the relevant key. Use the following command to generate the keystore containing the .crt and .key files:

```bash
openssl pkcs12 -export \
    -in <.crt file> -inkey <.key file> \
    -out truststore.p12
```


To make sure that the application can sign the HTTPS connection, we also need to add any root certificates. In my case I need to add the Dutch Government certificate which can be downloaded from http://cert.pkioverheid.nl/RootCA-G3.cer

```bash
keytool -import -alias rootCa_g3 -keystore truststore.p12 \
    -file RootCA-G3.cer
```

# 3. Configure RestTemplate
Place the keystorefile in the resources folder in your Spring application (preferably in a somewhat logical place). 

Add the file to your properties:
```bash
#trust store location
trust.store=classpath:truststore.p12
#trust store password
trust.store.password=<password>
```

Now we need to configure the RestTemplate bean in Spring. Use the following code example:

```java
@Bean
RestTemplate restTemplate() throws Exception {
    SSLContext sslContext = new SSLContextBuilder()
            .loadKeyMaterial(
                    createKeyMaterial(trustStore, trustStorePassword),
                    trustStorePassword.toCharArray())
            .loadTrustMaterial(trustStore.getURL(),
                    trustStorePassword.toCharArray())
            .build();
    SSLConnectionSocketFactory socketFactory =
            new SSLConnectionSocketFactory(sslContext);
    HttpClient httpClient = HttpClients.custom()
            .setSSLSocketFactory(socketFactory)
            .build();
    HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);
    return new RestTemplate(factory);
}

private KeyStore createKeyMaterial(Resource trustStore, String pwd)
        throws IOException, KeyStoreException, 
        CertificateException, NoSuchAlgorithmException {
    KeyStore keyStore = KeyStore.getInstance("PKCS12");
    File key = trustStore.getFile();
    try (InputStream in = new FileInputStream(key)) {
        keyStore.load(in, pwd.toCharArray());
    }
    return keyStore;
}
```

# 4. Make request!
To make a request, just use the RestTemplate bean!

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MutialSslTestApplicationTests {

	private static final String URL = "<URL>";

	@Autowired
	private RestTemplate restTemplate;

	@Test
	public void contextLoads() {
		String s = restTemplate.getForObject(URL, String.class);
		System.out.println(s);
	}
}
```
