FROM centos:centos6

#
#  Install devtools like Make and git
#
RUN yum groupinstall -y "Development Tools"
RUN yum install -y rpmdevtools
RUN yum install -y vim emacs nano
RUN yum install -y yum-utils

#
#  Add the EPEL repository for freetds and hiredis
#
RUN yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm

#
#  Install GCC-4.9 as it has the requisite support for C11 keywords and atomics
#
RUN yum install -y centos-release-scl
RUN yum install -y devtoolset-3-gcc devtoolset-3-gcc-c++
ENV CC=/opt/rh/devtoolset-3/root/usr/bin/gcc

#
#  Setup a src dir in /usr/local
#
RUN mkdir -p /usr/local/src/repositories
WORKDIR /usr/local/src

#
#  Get a modern version of cmake.  We need 3.8.2 or later to build libkqueue rpms
#
RUN yum install -y openssl

RUN curl https://cmake.org/files/v3.8/cmake-3.8.2-Linux-x86_64.sh > ./cmake.sh

RUN [ "$(cat ./cmake.sh | openssl sha256 | sed 's/^.* //')" = "bb26b1871f9e5c2fb73476186cc94d03b674608f704b48b94d617340b87b4d73" ]
RUN chmod +x ./cmake.sh
RUN ./cmake.sh --skip-license --prefix=/usr/local

#
#  Grab libkqueue and build
#
WORKDIR /usr/local/src/repositories
RUN git clone --depth=1 https://github.com/arr2036/libkqueue.git
WORKDIR libkqueue

# Fixme: Change to the main repo when they merge this branch
RUN git checkout patch-3
RUN cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib ./ && \
    make && \
    cpack -G RPM
RUN yum localinstall -y ./*.rpm

#
#  Use LTB's openldap packages intead of the distribution version to avoid linking against NSS
#
RUN echo $'[ltb-project]\n\
name=LTB project packages\n\
baseurl=https://ltb-project.org/rpm/$releasever/$basearch\n\
enabled=1\n\
gpgcheck=1\n\
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-LTB-project'\
> /etc/yum.repos.d/ltb-project.repo
RUN rpm --import https://ltb-project.org/lib/RPM-GPG-KEY-LTB-project


WORKDIR /usr/local/src/repositories
RUN git clone --depth=1 https://github.com/mattrose/hiredis.git
WORKDIR hiredis
RUN git checkout rpm
WORKDIR /usr/local/src/repositories
RUN mkdir -p $HOME/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
RUN yum-builddep -y hiredis/redhat/hiredis.spec
RUN v=$(rpm -q --qf '%{VERSION}\n' --specfile hiredis/redhat/hiredis.spec | head -1) ;\
    mv hiredis hiredis-$v ;\
    tar -c -z -f $HOME/rpmbuild/SOURCES/hiredis-$v.tar.gz --exclude=.git hiredis-$v ;\
    rpmbuild -bb hiredis-$v/redhat/hiredis.spec 
RUN yum localinstall -y $HOME/rpmbuild/RPMS/*/hiredis-*.rpm

#
#  Shallow clone the FreeRADIUS source
#
WORKDIR /usr/local/src/repositories
RUN git clone --depth=1 https://github.com/mattrose/freeradius-server.git

WORKDIR freeradius-server

# Install build dependencies for all branches
RUN for i in $(git for-each-ref --format='%(refname:short)' refs/remotes/origin 2>/dev/null | sed -e 's#origin/##' | egrep "^v[0-9]*\.[0-9x]*\.x$");\
	do \
		git checkout $i; \
		[ -e ./redhat/freeradius.spec ] && yum-builddep -y ./redhat/freeradius.spec; \
	done

# Create the build tree
ENV BUILDDIR=/root/rpmbuild
RUN rpmdev-setuptree
### This is necessary for the jenkins server to talk to the docker instance
RUN yum install -y openssh-server java-1.8.0-openjdk-headless createrepo
RUN adduser jenkins
RUN echo -e "jenkins1" | passwd --stdin jenkins
RUN mkdir /var/run/sshd
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ''
RUN sed -i 's|session    required     pam_loginuid.so|session    optional     pam_loginuid.so|g' /etc/pam.d/sshd
EXPOSE 22
CMD ["/usr/sbin/sshd","-D"]
