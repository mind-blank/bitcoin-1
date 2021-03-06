Release Process
====================

* update translations (ping wumpus, Diapolo or tcatm on IRC)
* see https://github.com/bitcoin/bitcoin/blob/master/doc/translation_process.md#syncing-with-transifex

* * *

###update (commit) version in sources

	contrib/verifysfbinaries/verify.sh
	doc/README*
	share/setup.nsi
	src/clientversion.h (change CLIENT_VERSION_IS_RELEASE to true)

###tag version in git

	git tag -s v(new version, e.g. 0.8.0)

###write release notes. git shortlog helps a lot, for example:

	git shortlog --no-merges v(current version, e.g. 0.7.2)..v(new version, e.g. 0.8.0)

* * *

###update gitian

 In order to take advantage of the new caching features in gitian, be sure to update to a recent version (e9741525c or higher is recommended)

###perform gitian builds

 From a directory containing the bitcoin source, gitian-builder and gitian.sigs
  
	export SIGNER=(your gitian key, ie bluematt, sipa, etc)
	export VERSION=(new version, e.g. 0.8.0)
	pushd ./bitcoin
	git checkout v${VERSION}
	popd
	pushd ./gitian-builder

###fetch and build inputs: (first time, or when dependency versions change)
 
	mkdir -p inputs; cd inputs/

 Register and download the Apple SDK: (see OSX Readme for details)
 
 https://developer.apple.com/downloads/download.action?path=Developer_Tools/xcode_4.6.3/xcode4630916281a.dmg
 
 Using a Mac, create a tarball for the 10.7 SDK and copy it to the inputs directory:
 
	tar -C /Volumes/Xcode/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/ -czf MacOSX10.7.sdk.tar.gz MacOSX10.7.sdk

 Build Bitcoin Core for Linux, Windows, and OS X:
  
	./bin/gbuild --commit bitcoin=v${VERSION} ../bitcoin/contrib/gitian-descriptors/gitian-linux.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-linux --destination ../gitian.sigs/ ../bitcoin/contrib/gitian-descriptors/gitian-linux.yml
	pushd build/out
	zip -r bitcoin-${VERSION}-linux-gitian.zip *
	mv bitcoin-${VERSION}-linux-gitian.zip ../../../
	popd
	./bin/gbuild --commit bitcoin=v${VERSION} ../bitcoin/contrib/gitian-descriptors/gitian-win.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-win --destination ../gitian.sigs/ ../bitcoin/contrib/gitian-descriptors/gitian-win.yml
	pushd build/out
	zip -r bitcoin-${VERSION}-win-gitian.zip *
	mv bitcoin-${VERSION}-win-gitian.zip ../../../
	popd
        ./bin/gbuild --commit bitcoin=v${VERSION} ../bitcoin/contrib/gitian-descriptors/gitian-osx.yml
        ./bin/gsign --signer $SIGNER --release ${VERSION}-osx --destination ../gitian.sigs/ ../bitcoin/contrib/gitian-descriptors/gitian-osx.yml
	pushd build/out
	mv Bitcoin-Qt.dmg ../../../
	popd
	popd

  Build output expected:

  1. linux 32-bit and 64-bit binaries + source (bitcoin-${VERSION}-linux-gitian.zip)
  2. windows 32-bit and 64-bit binaries + installer + source (bitcoin-${VERSION}-win-gitian.zip)
  3. OSX installer (Bitcoin-Qt.dmg)
  4. Gitian signatures (in gitian.sigs/${VERSION}-<linux|win|osx>/(your gitian key)/

repackage gitian builds for release as stand-alone zip/tar/installer exe

**Linux .tar.gz:**

	unzip bitcoin-${VERSION}-linux-gitian.zip -d bitcoin-${VERSION}-linux
	tar czvf bitcoin-${VERSION}-linux.tar.gz bitcoin-${VERSION}-linux
	rm -rf bitcoin-${VERSION}-linux

**Windows .zip and setup.exe:**

	unzip bitcoin-${VERSION}-win-gitian.zip -d bitcoin-${VERSION}-win
	mv bitcoin-${VERSION}-win/bitcoin-*-setup.exe .
	zip -r bitcoin-${VERSION}-win.zip bitcoin-${VERSION}-win
	rm -rf bitcoin-${VERSION}-win

**Mac OS X .dmg:**

	mv Bitcoin-Qt.dmg bitcoin-${VERSION}-osx.dmg

###Next steps:

Commit your signature to gitian.sigs:

	pushd gitian.sigs
	git add ${VERSION}-linux/${SIGNER}
	git add ${VERSION}-win/${SIGNER}
	git add ${VERSION}-osx/${SIGNER}
	git commit -a
	git push  # Assuming you can push to the gitian.sigs tree
	popd

-------------------------------------------------------------------------

### After 3 or more people have gitian-built and their results match:

- Perform code-signing.

    - Code-sign Windows -setup.exe (in a Windows virtual machine using signtool)

    - Code-sign MacOSX .dmg

  Note: only Gavin has the code-signing keys currently.

- Create `SHA256SUMS.asc` for the builds, and GPG-sign it:
```bash
sha256sum * > SHA256SUMS
gpg --digest-algo sha256 --clearsign SHA256SUMS # outputs SHA256SUMS.asc
rm SHA256SUMS
```
(the digest algorithm is forced to sha256 to avoid confusion of the `Hash:` header that GPG adds with the SHA256 used for the files)

- Upload zips and installers, as well as `SHA256SUMS.asc` from last step, to the bitcoin.org server

- Update bitcoin.org version

  - Make a pull request to add a file named `YYYY-MM-DD-vX.Y.Z.md` with the release notes
  to https://github.com/bitcoin/bitcoin.org/tree/master/_releases
   ([Example for 0.9.2.1](https://raw.githubusercontent.com/bitcoin/bitcoin.org/master/_releases/2014-06-19-v0.9.2.1.md)).

  - After the pull request is merged, the website will automatically show the newest version, as well
    as update the OS download links. Ping Saivann in case anything goes wrong

- Announce the release:

  - Release sticky on bitcointalk: https://bitcointalk.org/index.php?board=1.0

  - Bitcoin-development mailing list

  - Update title of #bitcoin on Freenode IRC

  - Optionally reddit /r/Bitcoin, ... but this will usually sort out itself

- Notify BlueMatt so that he can start building [https://launchpad.net/~bitcoin/+archive/ubuntu/bitcoin](the PPAs)

- Add release notes for the new version to the directory `doc/release-notes` in git master

- Celebrate 
