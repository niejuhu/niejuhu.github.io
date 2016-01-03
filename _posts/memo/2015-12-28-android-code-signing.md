Some notes about android code signing.

# AOSP apk signing

src: build/tools/signapk

bin: out/host/darwin-x86/framework/signapk.jar

Usage: signapk pubkey.x509.pem privkey.pkcs8.der input.apk output.apk

pubkey.x509.pem is the certificate file of x509 format in PEM encoding

privkey.pkcs8.der is the private key file of pkcs8 format in DER encoding

## Create keys

We can use the `make_key` in aosp to create keys used by signapk.

		$ evelopment/tools/make_key testkey  '/C=US/ST=California/L=Mountain \
		View/O=Android/OU=Android/CN=Android/emailAddress=android@android.com'

Here is the actual commands run by `make_key`:

		# Create the private key and the private key never touch the disk
		mknod ${one} p
		mknod ${two} p
		penssl genrsa -f4 2048 | tee ${one} > ${two}

		# Create self-signed x509 certificate file
		openssl req -new -x509 ${hash} -key ${two} -out $1.x509.pem \
		  -days 10000 -subj "$2" &

		# Create private key file in DER encoding, password is optional
		export password
		openssl pkcs8 -in ${one} -topk8 -outform DER -out $1.pk8 \
		  -passout env:password
		unset password

# Sign apk using JarSigner

Eclipse and other IDEs use jarsigner to sign apk.

Usage: jarsigner -keystore debug.keystore -sigalg SHA1withRSA -signedjar sign.apk test.apk debugkey

## Create keys

	$ keytool -genkey -alias debugkey -keystore debug.keystore -keyalg rsa

## Convert aosp keys to jarsigner keys

	# Convert der to pem
	$ openssl pkcs8 -in testkey.pk8 -inform DER -outform PEM -out testkey.priv.pem -nocrypt
	# Export to keystore in pkcs12 keystore type
	$openssl pkcs12 -export -in testkey.x509.pem -inkey testkey.priv.pem -out testkey.pk12 \
		-name testkey
	# Import to keystore in jks keystore type that is the keytool's default keystore type
	$ keytool -importkeystore -destkeystore debug.keystore -srckeystore test.pk12 -srcstoretype \
		PKCS12 -alias testkey

# Process of signing apk

Given an unsigned apk file that has the following conent:

		$ unzip -l unsig.apk  | head -20
		Archive:  unsig.apk
		  Length     Date   Time    Name
		 --------    ----   ----    ----
			 1832  09-24-15 13:08   AndroidManifest.xml
		  2636764  09-24-15 13:08   classes.dex
				0  01-03-16 11:37   res/
				0  01-03-16 11:37   res/anim/
			  396  09-24-15 13:08   res/anim/abc_fade_in.xml

After signing using one private key called testkey2, its content becomes:

		$ unzip -l sig.apk  | head -20
		Archive:  sig.apk
		  Length     Date   Time    Name
		 --------    ----   ----    ----
			35703  01-03-16 12:40   META-INF/MANIFEST.MF
			35865  01-03-16 12:40   META-INF/TESTKEY2.SF
			 1325  01-03-16 12:40   META-INF/TESTKEY2.RSA
			 1832  09-24-15 13:08   AndroidManifest.xml
		  2636764  09-24-15 13:08   classes.dex
				0  01-03-16 11:37   res/
				0  01-03-16 11:37   res/anim/
			  396  09-24-15 13:08   res/anim/abc_fade_in.xml

The apk file is signed in 3 steps:

1. Calculate digest of each file and write it to the META-INF/MANIFEST.MF. Take the
AndroidManifest.xml as example the equivalent openssl command:

		$ openssl dgst -sha256 -binary unsig/AndroidManifest.xml | openssl base64
		wn5oDPLnJAO5ixTLnxRbb7nZnvFW3ehGYYUN1jt1oCc=

2. Calculate digests of the META-INF/MANIFEST.MF file and its\' entries and write them to
META-INF/CERT.SF.  The equivalent openssl commands:

		$ openssl dgst -sha256 -binary sig/META-INF/MANIFEST.MF | openssl base64
		$ echo -en "Name: AndroidManifest.xml\r\nSHA-256-Digest: \
			wn5oDPLnJAO5ixTLnxRbb7nZnvFW3ehGYYUN1jt1oCc=\r\n\r\n" |\
			openssl dgst -sha256 -binary | openssl base64

3. Calculate signature of META-INF/CERT.SF and write it with the signing certificate to the
META-INF/CERT.KEYALG in PKCS7 format in DER encoding. You can use the openssl asn1parse
command to view the signature and use openssl smime command to verify the signature.

		$ openssl asn1parse -inform DER -in sig/META-INF/CERT.RSA -i
		0:d=0  hl=4 l=1720 cons: SEQUENCE
		4:d=1  hl=2 l=   9 prim:  OBJECT            :pkcs7-signedData
		...

		# -noverify : do not verify signing certificate
		$ openssl smime -verify -in sig/META-INF/CERT.RSA -inform DER -content \
			sig/META-INF/CERT.SF -noverify testkey.x509.pem
		ignature-Version: 1.0
		Created-By: 1.0 (Android SignApk)
		SHA1-Digest-Manifest: cECRsXRTc2jXSvYJnMp48XsfkGg=
		...
		Verification successful

# verify signature

		$ jarsigner -keystore testkey2.ks -verify sig2.apk -certs -verbose

# Dump certificate from signed apk

Some times you want to view or dump to a file the signing certificate. Here is the commands:

		# View from apk file.
		$ keytool -printcert -jarfile sig.apk
		# View from .RSA file.
		$ keytool -printcert -file sig/META-INF/CERT.RSA

		# Dump from apk file. **NOTE** The output file has 2 header lines describing subject and
		# issuer that maybe you should delete for compatible.
		$ unzip -c -q sig.apk META-INF/CERT.RSA | openssl pkcs7 -inform DER -print_certs \
			-out cert.pem
		# Dump from .RSA file.
		$ openssl pkcs7 -in sig/META-INF/CERT.RSA -inform DER -print_certs -out cert.pem

**NOTE** The output file has 2 header lines describing subject and issuer that maybe you should
delete for compatible.
