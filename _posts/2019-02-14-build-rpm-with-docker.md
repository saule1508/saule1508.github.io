---
layout: post
title: Build a rpm with docker
published: false
---

When I have to build a rpm I generally make a jenkins job that calls a build.sh script to do the rpm. It is convenient to use a docker image so that the jenkins agent is not polluted. The docker image is created on the fly then immediately used. Here is how my build script will look like

Since I am using jenkins, the variable BUILD_NUMBER and WORKSPACE are set for me.

The version is build from the file build/version.txt (it is a convention I use in my projects) to which BUILD_NUMBER is appended. However when I build on develop I append snapshot instead. So if version.txt contains 7.03 and if I am building on develop, the rpm built will have version 7.03-snapshot.

``` bash
help(){
 echo "$0 -b <branch> where branch is develop for example. When branch is develop, snapshot will be added to the version"
 exit 1
}

while getopts b:h ARG
do
   case $ARG in
      b ) BRANCH=${OPTARG};;
      h ) help ;;
      * ) echo "invalid parameter"
          help
          exit;;
   esac
done
VERSION=$(cat build/version.txt)
if [ $BRANCH == "develop" ] ; then
  BUILD_NUMBER="snapshot"
fi
if [ -d rpms ] ; then
 sudo rm -rf rpms
fi
mkdir rpms
chmod 777 rpms
echo VERSION is $VERSION and BUILD_NUMBER is $BUILD_NUMBER
docker build -t oraevs-build .
if [ -z $WORKSPACE ] ; then
  WORKSPACE=$(pwd)
fi
docker run --rm=true -v ${WORKSPACE}/rpms:/rpms -e BUILD_NUMBER=${BUILD_NUMBER} -e VERSION=$VERSION --user rpmbuild oraevs-build /home/rpmbuild/buildoraevs.bash

```
The artifact (the rpm build) will be stored in rpms directory

My Dockerfile contains this


```bash
FROM centos:centos7
MAINTAINER ptim007@yahoo.com
RUN yum -y install rpm-build redhat-rpm-config make gcc git vi tar unzip rpmlint && yum clean all
RUN useradd rpmbuild -u 5002 -g users -p rpmbuild
# here I add everything I need to make the rpm: sources, the build script
ADD build /home/rpmbuild/build
ADD dbautils /home/rpmbuild/dbautils
ADD orautils /home/rpmbuild/orautils
ADD capmgmt /home/rpmbuild/capmgmt
USER rpmbuild
ENV HOME /home/rpmbuild
RUN mkdir -p /home/rpmbuild/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
RUN echo '%_topdir %{getenv:HOME}/rpmbuild' > /home/rpmbuild/.rpmmacros

```

The buildoraevs.bash script will typically execute the rpmbuild command

```bash
#!/bin/bash
#

while getopts v:h: ARG
do
   case $ARG in
      v ) VERSION=$OPTARG;;
      h ) help ;;
      * ) echo "invalid parameter"
          help
          exit;;
   esac
done

if [ -z $BUILD_NUMBER ] ; then
 echo BUILD_NUMBER is not know, it is normally given by jenkins
 exit 1
fi
if [ -z $VERSION ] ; then
  echo VERSION is not set via parameter neither via ENV, will use version.txt
  VERSION=`cat ./build/version.txt`
fi

cd /home/rpmbuild
if [ ! -f ./dbautils/build/evs-mad-oraevs.spec ] ; then
 echo Sorry, can not find file ./dbautils/build/evs-mad-oraevs.spec
 exit 1
fi
cp ./dbautils/build/evs-mad-oraevs.spec $HOME/rpmbuild/SPECS

# prepare a tar.gz file according to your spec file and copy it  to the SOURCES directory
...
tar -zcf evs-mad-oraevs-${VERSION}.tar.gz evs-mad-oraevs-${VERSION}
cp evs-mad-oraevs-${VERSION}.tar.gz $HOME/rpmbuild/SOURCES/
# then execute the rpmbuild command
cd $HOME/rpmbuild
rpmbuild -ba --define "_buildnr ${BUILD_NUMBER}" --define "_myversion $VERSION" ./SPECS/evs-mad-oraevs.spec
# copy the rpms to the artifact directory, for jenkins.
if [[ -d /rpms ]] ; then
 cp ./RPMS/noarch/evs-mad-oraevs*.rpm /rpms/
 
```


