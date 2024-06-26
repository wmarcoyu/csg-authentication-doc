# Configuring SimpleSAMLphp "to print hello world"
This documentation contains instructions for setting up a minimal SimpleSAMLphp
demonstration that allows users to sign in with SimpleSAMLphp being both the 
service provider (SP) and the identity provider (IdP).

## 1. Basic Configuration
### Download and install
Go to the install directory. `/var/` is the default and requires the least amount 
of extra configuration.
```
cd /var/
```
Download (https://simplesamlphp.org/download/), 
unzip, and rename the directory to `simplesamlphp`.
```
tar xzf simplesamlphp-x.y.z.tar.gz
mv simplesamlphp-x.y.z simplesamlphp
```

### Configure Apache
This assumes that simplesamlphp is installed under `/var/`.

Open `/etc/apache2/sites-available/000-default.conf` with a text editor and edit 
the configuration to the following.
```
<VirtualHost *>
    ServerName service.example.com
    DocumentRoot /var/www/service.example.com

    SetEnv SIMPLESAMLPHP_CONFIG_DIR /var/simplesamlphp/config

    Alias /simplesaml /var/simplesamlphp/public

    <Directory /var/simplesamlphp/public>
        Require all granted
    </Directory>
</VirtualHost>
```

### Edit `config.php`
Directory `simplesamlphp/config/` contains the basic configuration files. Open 
`config.php` with a text editor.

Set `'baseurlpath'` to **exactly** the following so that it's consistent with the 
`Alias` entry in the Apache config file.
```
'baseurlpath' => 'simplesaml/',
```

Set an admin password. The default value is `'123'` and it must be changed.
```
'auth.adminpassword' => 'setnewpasswordhere',
```

Generate a random secret salt. For example, run the following command in terminal
```
openssl rand -base64 32
```
and copy paste the generated salt here.
```
'secretsalt' => 'generatedsecretsalt',
```

Set a technical contact name and email.
```
'technicalcontact_name' => 'John Smith',
'technicalcontact_email' => 'john.smith@example.com',
```

Now you should be able to see a welcome page at 
`https://service.example.org/simplesaml/` and an admin dashboard at 
`https://service.example.org/simplesaml/admin/`.

Source of this section: 
`https://simplesamlphp.org/docs/stable/simplesamlphp-install.html`.
Check there for more optional settings.

## 2. Service Provider (SP) Configuration
The SP configuration is in `config/authsources.php`.

Use `example-userpass` as the IdP for the hello world demonstration.
```
'example-userpass' => [
        'exampleauth:UserPass',

        // Give the user an option to save their username for future login attempts
        // And when enabled, what should the default be, to save the username or not
        //'remember.username.enabled' => false,
        //'remember.username.checked' => false,

        'users' => [
            'student:studentpass' => [
                'uid' => ['test'],
                'eduPersonAffiliation' => ['member', 'student'],
            ],
            'employee:employeepass' => [
                'uid' => ['employee'],
                'eduPersonAffiliation' => ['member', 'employee'],
            ],
        ],
    ],
```

Add IdP metadata to the SP so that the SP is aware of the IdP.
Edit `metadata/saml20-idp-remote.php` as follows.
```
<?php
$metadata['https://example.org/saml-idp'] = [
    'SingleSignOnService'  => 'https://example.org/simplesaml/saml2/idp/SSOService.php',
    'SingleLogoutService'  => 'https://example.org/simplesaml/saml2/idp/SingleLogoutService.php',
    'certificate'          => 'example.org.pem',
];
```
NOTE: `example.org.pem` is the certificate of the IdP, 
not of the SP itself. It should be placed under the `cert/` directory.

Source of this section:
`https://simplesamlphp.org/docs/stable/simplesamlphp-sp.html`.
Check there for more optional settings and using other IdPs.

## 3. Identity Provider (IdP) Configuration
First enable saml20 by editing `config/config.php`.
```
'enable.saml20-idp' => true,
```
Then enable exampleauth also in `config/config.php`.
```
'module.enable' => [
    'exampleauth' => true,
    …
],
```

Create a self-signed certificate for the IdP.
```
openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes -out example.org.crt -keyout example.org.pem
```

Configure the IdP itself in `metadata/saml20-idp-hosted.php`.
```
<?php

$metadata['https://example.org/saml-idp'] = [
    /*
     * The hostname for this IdP. This makes it possible to run multiple
     * IdPs from the same configuration. '__DEFAULT__' means that this one
     * should be used by default.
     */
    'host' => '__DEFAULT__',

    /*
     * The private key and certificate to use when signing responses.
     * These can be stored as files in the cert-directory or retrieved
     * from a database.
     */
    'privatekey' => 'example.org.pem',
    'certificate' => 'example.org.crt',

    /*
     * The authentication source which should be used to authenticate the
     * user. This must match one of the entries in config/authsources.php.
     */
    'auth' => 'example-userpass',
];
```
Also make sure the following is uncommented in the file to use the 
`urn:oasis:names:tc:SAML:2.0:attrname-format:uri` NameFormat.
```
'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',
'authproc' => [
    // Convert LDAP names to oids.
    100 => ['class' => 'core:AttributeMap', 'name2oid'],
],
```

Add the SP data to the IdP so that the IdP is aware of the SP. Edit 
`metadata/saml20-sp-remote.php` as follows.
```
<?php

$metadata['https://sp.example.org/simplesaml/module.php/saml/sp/metadata.php/default-sp'] = [
    'AssertionConsumerService' => 'https://sp.example.org/simplesaml/module.php/saml/sp/saml2-acs.php/default-sp',
    'SingleLogoutService' => 'https://sp.example.org/simplesaml/module.php/saml/sp/saml2-logout.php/default-sp',
];
```

Now there should be a login page at `https://service.example.org/simplesaml/admin/`
under `test` and `example-userpass`. The user information is defined at 
`config/authsources.php`, which by default has two users `student` and `employee`
with passwords being `studentpass` and `employeepass`.

Source of this section:
`https://simplesamlphp.org/docs/stable/simplesamlphp-idp.html`.

## Issues and TODOs
* Ideally, `simplesamlphp` should be installed under the `/var/` directory. It is 
the simplest although other installation paths would work with extra configuration.
However, accessing the `/var/` directory requires either a traditional host server
with a Linux file system or a Docker container, both of which are not 
easily achievable, if achievable at all, with Pantheon.
* Similar to the first bullet point, Pantheon makes it difficult, if possible at 
all, to configure Apache. We cannot locate the config file and Pantheon's 
philosophy does not allow users to configure Apache.
* Using a production value IdP could be a hassle. The best solution is InCommon,
which has IdP metadata of each participating university. Coordination with 
InCommon is required, although a better question would be why we need authentication
and if we can only grant priviledged access to a handful of administrators, 
meaning that we could use the example-userpass approach. It's not entirely bad
since there is a centralized point of management.
