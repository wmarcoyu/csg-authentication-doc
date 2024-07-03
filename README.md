# Configuring SimpleSAMLphp with UMich IdP
This documentation contains instructions for setting up a minimal SimpleSAMLphp
demonstration that allows users to sign in with SimpleSAMLphp as the service provider
(SP) and with the University of Michigan as the identity provider (IdP). 

:warning: **The instructions are specifically for WordPress on Pantheon.**

## 1. Basic Configuration
### Download and install
Go to Pantheon dashboard -> Database / Files -> Import -> Archiver of site files. 
Select URL and copy-paste the download link for the latest version that can be 
found here https://simplesamlphp.org/download/ (not this link itself). Click import.

After the import is finished, simplesamlphp will be downloaded and unzipped under
`/files/` on the SFTP server, which can be accessed with a GUI by clicking Code ->
Connect with SFTP -> Open SFTP client on the dashboard. There will be a 
simplesamlphp-x.y.z folder under `/files/` with x.y.z being a version number. As of 
July 3, 2024, the latest version is 2.2.2.

The main project directory of the application is at `/code/`.

### Create a `simplesaml` Link
First log into the server with SFTP at command line. Again go to the Pantheon 
dashboard, click Connection Info and copy the command line under SFTP. Paste the 
command into a terminal and run it.

Having logged into the server, run
```
ln -s /files/simplesamlphp-x.y.z/public /code/simplesaml
```
This exposes the public folder of simplesamlphp to the application.

NOTE: avoid moving files and directories in SFTP servers, which is both complicated
and slow.

### Edit `config.php`
Directory `/files/simplesamlphp-x.y.z/config/` 
contains the basic configuration files. Open `config.php` with a text editor.
You may need to run `rename config.php.dist config.php` first to remove the `.dist`
suffix.

Set `'baseurlpath'` to the following. Make sure to always use https.
```
'baseurlpath' => 'https://<website-name>/simplesaml/',
```

Set an admin password. The default value is `'123'` and it must be changed.
```
'auth.adminpassword' => 'setnewpasswordhere',
```

Generate a random secret salt. For example, run the following command in another
"traditional" unix terminal that is not connected to the SFTP server.
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
`https://<website-name>/simplesaml/` and an admin dashboard at 
`https://<website-name>/simplesaml/admin/`.

#### If you can't see the welcome page
If there is an error in loading the welcome page (simplesaml/) and the error message
says that the `'cachedir'` entry is invalid, it's typically because you do not have 
write access to the default cache directory. Make the following change 
in `config.php`.
```
...
'cachedir' => '/files/simplesamlphp-2.2.2/cache/',  // or any folder not owned by root
...
```
Pantheon does not allow users to change the ownership of default directories.

#### If you can't see the admin page
If you are able to see the welcome page but not the admin page, and it gives a 
NOSTATE error, make the following change in `config.php`.

```
...
'session.phpsession.cookiename' => 'PHPSESSID',  // originally this is 'SimpleSAML'
...
```
This setting attempts to override the default setting in `php.ini`, which results in
inconsistent cookie names. Changing the value back to the default 'PHPSESSID'
solves the issue.

Source of this section: 
`https://simplesamlphp.org/docs/stable/simplesamlphp-install.html`.
Check there for more optional settings.

## 2. Service Provider (SP) Configuration
### Generate Certificates for the SP
Edit the following entry in `config/config.php`.
```
...
'certdir' => '/files/simplesamlphp-x.y.z/cert/',
...
```
Go to the directory (either in GUI or terminal) and create two files, 
`sp.pem` and `sp.crt`.
(Their names can be arbitrary, sp meaning service providers, but they have to 
match the following command)

Then go to a "traditional" unix terminal that is not connected to the SFTP server,
and run
```
openssl req -newkey rsa:3072 -new -x509 -days 3652 -nodes -out sp.crt -keyout sp.pem
```
and follow the prompts.

:warning: Note the match in output filenames.

Copy the content of `sp.pem` and `sp.crt` in the local file system to 
the `sp.pem` and `sp.crt` in the remote SFTP file system, respectively.

### Configure the SP
The SP configuration is in `config/authsources.php`. Remove the `.dist` suffix if 
necessary.

Edit the following entries inside `default-sp`.
```
...
'entityID' => 'https://<website-name>/',
'idp' => 'https://shibboleth.umich.edu/idp/shibboleth',
'privatekey' => 'sp.pem',
'certificate' => 'sp.crt',
...
```
Again notice the filenames for `privatekey` and `certificate` should match the ones
you set earlier.

Source of this section:
`https://simplesamlphp.org/docs/stable/simplesamlphp-sp.html`.
Check there for more optional settings and using other IdPs.

## 3. Identity Provider (IdP) Configuration

Add IdP metadata to the SP so that the SP is aware of the IdP.
Edit `metadata/saml20-idp-remote.php` as follows.
```
 $metadata['https://shibboleth.umich.edu/idp/shibboleth'] = [
    'SingleSignOnService'  => 'https://weblogin.umich.edu/idp/profile/SAML2/Redirect/SSO',
    'SingleLogoutService'  => 'https://weblogin.umich.edu/idp/profile/SAML2/Redirect/SLO',
    'certificate'          => 'umich-idp.pem',
];
```
This is the University of Michigan IdP metadata retrieved from here, 
https://shibboleth.umich.edu/idp/shibboleth

To add metadata for other instituions, see this list, 
https://incommon.org/custom/federation/info/all-entity-categories.html

Now there should be a login page at `https://service.example.org/simplesaml/admin/`
under `test` and `default-sp`. The password for logging into the admin page is 
`auth.adminpassword` defined earlier in `config.php`. Click on `default-sp` and it 
should redirect you to the University of Michigan single sign-on (SSO) page. Seeing
the UM page means that the configuration so far is correct. It will give you an error
(Unsupported Request: The application you have accessed is not registered for 
use with this service)
because you haven't sent your (service provider's) metadata to UM. To obtain the 
SP's metadata, go to the Federation tab on the admin page.

Source of this section:
`https://simplesamlphp.org/docs/stable/simplesamlphp-idp.html`.
