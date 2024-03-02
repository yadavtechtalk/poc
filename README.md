# poc
@SpringBootApplication(exclude = {MongoAutoConfiguration.class, MongoDataAutoConfiguration.class})
public class YourApplication {
    public static void main(String[] args) {
        SpringApplication.run(YourApplication.class, args);
    }
}


public SSLContext createSSLContext() throws Exception {
    // Load the PEM file
    ClassPathResource resource = new ClassPathResource("yourCertificate.pem");
    InputStream is = resource.getInputStream();
    PEMParser pemParser = new PEMParser(new InputStreamReader(is));
    PEMKeyPair pemKeyPair = (PEMKeyPair) pemParser.readObject();
    KeyPair kp = new JcaPEMKeyConverter().getKeyPair(pemKeyPair);
    PrivateKey privateKey = kp.getPrivate();
    JcaX509CertificateConverter certConverter = new JcaX509CertificateConverter();
    X509Certificate certificate = certConverter.getCertificate(pemParser.readObject());

    // Create a KeyStore and add the certificate and private key
    KeyStore keystore = KeyStore.getInstance(KeyStore.getDefaultType());
    keystore.load(null); // You don't need the KeyStore instance to come from a file.
    keystore.setKeyEntry("alias", privateKey, "".toCharArray(), new Certificate[]{certificate});

    // Initialize the SSLContext
    KeyManagerFactory keyFac = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
    keyFac.init(keystore, "".toCharArray());
    SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
    sslContext.init(keyFac.getKeyManagers(), null, null);
    return sslContext;
}


@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {
    @Override
    public MongoClient mongoClient() {
        ConnectionString connectionString = new ConnectionString("mongodb://your-host:port/database");
        SSLContext sslContext = null;
        try {
            sslContext = createSSLContext();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
        MongoClientSettings settings = MongoClientSettings.builder()
                .applyConnectionString(connectionString)
                .applyToSslSettings(builder -> {
                    builder.enabled(true);
                    builder.context(sslContext);
                    builder.invalidHostNameAllowed(true);
                })
                .build();
        return MongoClients.create(settings);
    }
}
