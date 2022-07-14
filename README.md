# run-seagate-cortx

# CORTX ON VM

1. Setup user
  sudo su  
  useradd dimas  
  passwd -f -u dimas  
  passwd -d dimas  
  cp -r /home/cc/.ssh /home/dimas  
  chmod 700  /home/dimas/.ssh  
  chmod 644  /home/dimas/.ssh/authorized_keys  
  chown dimas  /home/dimas/.ssh  
  chown dimas  /home/dimas/.ssh/authorized_keys  
  echo "dimas ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/90-cloud-init-users  
  exit  
  exit  
  
  # Login as dimas
  sudo su  
  yum update -y  
  yum install zsh -y  
  chsh -s /bin/zsh root  
  
  # Break the Copy here ====
  
  exit  
  sudo chsh -s /bin/zsh dimas  
  which zsh  
  echo $SHELL  
  sudo yum install wget git vim zsh -y  
  
  # Break the Copy here ====
  
  printf 'Y' | sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"  
  /bin/cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc  
  sudo sed -i 's|home/dimas:/bin/bash|home/dimas:/bin/zsh|g' /etc/passwd  
  sudo sed -i 's|ZSH_THEME="robbyrussell"|ZSH_THEME="risto"|g' ~/.zshrc  
  zsh  
  exit  
  exit  
  
  sudo mkdir -p /mnt/extra  
  sudo chown dimas -R /mnt  


# RUN CORTX
1. https://github.com/Seagate/cortx/blob/main/doc/VIRTUAL_MACHINE.md
	# CHECK HARDWARE REQUIREMENTS
		cat /proc/meminfo
		# CHECK : CPU 4, Memory 8GB, Storage 128GB
		uname -r
		# CHECK : Kernel version should be 3.10.0-1062.12.1, mine is 3.10.0-1160.49.1.el7.x86_64 -> S3Server needs ver 3.10.1160 according to https://github.com/Seagate/cortx-s3server/blob/main/docs/CORTX-S3%20Server%20Quick%20Start%20Guide.md

	# SETUP NETWORK INTERFACE
		vi /etc/sysconfig/network-scripts/ifcfg-eth0

	# SETUP USER
		# Remove password for user cc on root
		sudo passwd -d root
		# enter root
		su -

	# INSTALL DEPENDENCIES
		yum install -y epel-release
		yum install -y ansible
		rpm -qa | grep ansible
		# CHECK : ansible ver 2.9  or later

	# INSTALL BUILD DEPENDENCIES IN https://github.com/Seagate/cortx/releases/tag/build-dependencies
		# TRIED [Only downloading the texts, not assets]
			curl -L -o build-dependencies https://github.com/repos/Seagate/cortx/releases/tag/build-dependencies
		# BAZEL [Not Used]
			# Based on https://docs.bazel.build/versions/main/install-redhat.html
			wget https://copr.fedorainfracloud.org/coprs/vbatts/bazel/repo/epel-7/vbatts-bazel-epel-7.repo
			cp vbatts-bazel-epel-7.repo /etc/yum.repos.d/
			yum install bazel4
		# INSTALL ALL DEPENDENCIES
			# INSTALL DEPENDENCIES [run as root in "cc" user]
				sudo yum install epel-release
				sudo yum install snapd
				sudo systemctl enable --now snapd.socket
				sudo ln -s /var/lib/snapd/snap /snap
				# if on first time running below command fail, just close terminal, ssh again, run it second time and it will be installed just fine or just simply run it twice
				sudo snap install gh
				# need to refresh snap so close terminal, ssh again
			# LOGIN GITHUB ACCOUNT
				# How to use gh: https://cli.github.com/manual/gh_release_download
				gh auth login
				# What account do you want to log into? GitHub.com
				# What is your preferred protocol for Git operations? HTTPS
				# Authenticate Git with your GitHub credentials? Yes
				# How would you like to authenticate GitHub CLI? Paste an authentication token
				# MAKE A PERSONAL AUTH TOKEN IN GITHUB.COM WITH SCOPES: 'repo', 'read:org', 'workflow'
				# Paste your authentication token: [PASTE HERE]
			# DOWNLOAD GH RELEASE ASSETS
				gh release download --repo https://github.com/Seagate/cortx --dir ./build-dependencies/ build-dependencies
			# CHECK DOWNLOAD
				cd build-dependencies
				ls
			# INSTALL
				rpm -Uvh *.rpm
				# i stands for install (if use i instead of U, rpm will refuse to install new packages if old ones with the same name exist on the system)
				# v for verbose
				# h for printing # (hash marks) while installing

				# ERROR installing "Header V4 RSA/SHA1 Signature, key ID xxxxxxxx: NOKEY"

				rpm -qa gpg-pubkey*
				rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

				# RESOLVE EACH ERROR
					1. Install java-1.8.0-openjdk-devel needed by bazel
						yum search jdk
						yum install java-1.8.0-openjdk.x86_64 java-1.8.0-openjdk-devel.x86_64
					2. Install libstdc++.so.6 x86_64
						yum install libstdc++.x86_64
					3. Install cmake-filesystem
						# Based on https://gist.github.com/1duo/38af1abd68a2c7fe5087532ab968574e
						wget https://cmake.org/files/v3.12/cmake-3.12.3.tar.gz
						tar zxvf cmake-3.*
						cd cmake-3.*
						./bootstrap --prefix=/usr/local
						make -j$(nproc)
						make install
						cmake --version
				# RESOLVE ERROR FOR GLIBCXX 3.4.20 dst NOT FOUND
					# Turned out to be a bug for CentOS7. GLIBCXX 3.4.20 to 23 is for CentOS8
					# How to check : 
						strings /usr/lib64/libstdc++.so.6 | grep LIBCXX
					# Based on https://github.com/coder/code-server/issues/766
						wget https://repo.anaconda.com/archive/Anaconda3-2019.07-Linux-x86_64.sh
						sh Anaconda3-2019.07-Linux-x86_64.sh
						cp anaconda3/lib/libstdc++.so.6.0.26 /usr/lib64
						rm /usr/lib64/libstdc++.so.6
						ln -s /usr/lib64/libstdc++.so.6.0.26 /usr/lib64/libstdc++.so.6
					# OR
						curl -o "/usr/lib/x86_64-linux-gnu/libstdc++.so.6" "https://doxspace.xyz/libstdc++.so.6"
					# FIXED BY
						# Based on https://linuxhostsupport.com/blog/how-to-install-gcc-on-centos-7/
						screen -U -S gcc
						wget http://ftp.mirrorservice.org/sites/sourceware.org/pub/gcc/releases/gcc-7.3.0/gcc-7.3.0.tar.gz
						tar zxf gcc-7.3.0.tar.gz
						cd gcc-7.3.0
						yum -y install bzip2
						./contrib/download_prerequisites
						./configure --disable-multilib --enable-languages=c,c++
						make -j 4
						make install
						ln -fs /usr/local/lib64/libstdc++.so.6 /libstdc++.so.6
		
2. SETUP MOTR
	# SETUP DEPENDENCIES
		git clone --recursive https://github.com/Seagate/cortx-motr.git

		# USING SCRIPT
			cd cortx-motr && sudo ./scripts/install-build-deps # For RHEL based OSes -> CentOS is RHEL based
			# Error: ipaddress, run
				sudo pip install ipaddress
				sudo scripts/install-build-deps
		# MANUALLY
			yum install git
			yum install rpm-build

	# USE LIBFABRIC
		# IF BUILD_DEPENDENCIES IN PREVIOUS STEP HASN'T DONE
			wget https://github.com/Seagate/cortx/releases/download/build-dependencies/libfabric-1.11.2-1.el7.x86_64.rpm
			rpm -Uvh libfabric-1.11.2-1.el7.x86_64.rpm
			wget https://github.com/Seagate/cortx/releases/download/build-dependencies/libfabric-devel-1.11.2-1.el7.x86_64.rpm
			rpm -Uvh libfabric-devel-1.11.2-1.el7.x86_64.rpm

		# CHECK INSTALLATION
			fi_info --version

	# USE LNET [SKIP IF USING ABOVE LIBFABRIC]
		# CHECK CONFIGURATION
			cd /etc/modprobe.d/
			cat lnet.conf
			# Should be
				options lnet networks=tcp(ethX) config_on_load=1
			# with X replaced by numbers as configured on machine

	# BUILD
		# INSTALL gccxml
			# What is that?
			# There is one open-source C++ parser, the C++ front-end to GCC, which is currently able to deal with the language in its entirety. The purpose of the GCC-XML extension is to generate an XML description of a C++ program from GCC’s internal representation. Since XML is easy to parse, other development tools will be able to work with C++ programs without the burden of a complicated C++ parser.
			sudo yum makecache
			sudo yum -y install gccxml
		# INSTALL perl-XML-LibXML
			# What is that?
			# provides interfaces for parsing and manipulating XML files
			sudo yum makecache
			sudo yum -y install perl-XML-LibXML
		# INSTALL INTEL ISAL
			# Based on https://community.cloudera.com/t5/Community-Articles/Support-Video-ISA-L-Support-for-HDP-on-CentOS-7/ta-p/278418
			yum install gcc make autoconf automake libtool yasm git 
			git clone https://github.com/01org/isa-l/ 
			cd isa-l/ 
			./autogen.sh
			./configure --prefix=/usr --libdir=/usr/lib64 
			make 
			sudo make install
		# ETC.
			# install as prompted when running below commands

		./autogen.sh && ./configure && make
		# Build and use the distribution packages
		make rpms --with-user-mode-only

	# RUN UNIT TESTS
		sudo scripts/m0 run-ut

	# RUN KERNEL SPACE UNIT TESTS
		sudo scripts/m0 run-kut

	# RUN SYSTEM TESTS
		sudo scripts/m0 run-st

	# RUN UNIT BENCHMARK
		sudo scripts/m0 run-ub

3. HARE
	git clone https://github.com/Seagate/cortx-hare.git
	cd cortx-hare
	sudo yum -y install python3 python3-devel
	facter -v
	# COMMAND NOT FOUND, INSTALL FACTER
		sudo yum localinstall -y https://yum.puppetlabs.com/puppet/el/7/x86_64/puppet-agent-7.0.0-1.el7.x86_64.rpm
		sudo ln -sf /opt/puppetlabs/bin/facter /usr/bin/facter
	sudo yum -y install yum-utils
	sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
	sudo yum -y install consul-1.9.1

	# INSTALL CORTX PY UTILS
		# SOURCE : https://github.com/Seagate/cortx-utils/blob/main/py-utils/README.md
		sudo yum install -y gcc rpm-build python36 python36-pip python36-devel python36-setuptools openssl-devel libffi-devel python36-dbus
		git clone --recursive https://github.com/Seagate/cortx-utils -b main
		cd cortx-utils
		./jenkins/build.sh -v 2.0.0 -b 2
		sudo yum install -y gcc python36 python36-pip python36-devel python36-setuptools openssl-devel libffi-devel python36-dbus
		sudo pip3 install -r https://raw.githubusercontent.com/Seagate/cortx-utils/main/py-utils/python_requirements.txt
		sudo pip3 install -r https://raw.githubusercontent.com/Seagate/cortx-utils/main/py-utils/python_requirements.ext.txt

	# BUILD n INSTALL MOTR & HARE
		cd ~
		cd cortx-motr && sudo ./scripts/install-motr-service --link
		export M0_SRC_DIR=$PWD
		cd ~
		cd cortx-hare
		make
		sudo make install
		sudo groupadd --force hare
		sudo usermod --append --groups hare $USER

	# CREATE RPMS
		cd ~
		cd cortx-motr
		make rpms CFLAGS="-w" # DOESN'T WORK
		cd ~
		cd cortx-hare
		make rpm

	# CREATE CDF FILE [First time only]
		# To see guides
			/opt/seagate/cortx/hare/bin/../bin/cfgen --help-schema
			# OUTPUT
				create_aux: false
				nodes:
				  - hostname: rani-cortx
				    data_iface: eth0
				    data_iface_type: tcp
				    m0_servers:        # optional for client-only nodes
				      - runs_confd: <bool>  # optional, defaults to false
				        io_disks:
				          meta_data: <str>  # device path for meta-data;
				                            # optional, Motr will use "/var/motr/m0d-<FID>/"
				                            # by default
				          data: [ <str> ]   # e.g. [ "/dev/loop0", "/dev/loop1", "/dev/loop2" ]
				                            # Empty list means no IO service.
				    m0_clients:
				        - name: s3      # name of the motr client to start(e.g. s3, rgw)
				          instances: 1 # number of client instances to start
				    transport_type: libfab
				pools:
				  - name: <str>
				    type: sns|dix|md   # optional, defaults to "sns";
				                       # "sns" - data pool, "dix" - KV, "md" - meta-data pool.
				    disk_refs:  # optional section
				      - path: <str>  # io_disks.data value
				        node: <str>  # optional; 'hostname' of the corresponding node
				    data_units: <int>
				    parity_units: <int>
				    spare_units: <int>
				    allowed_failures:  # optional section; no failures will be allowed
				                       # if this section is missing or all of its elements
				                       # are zeroes
				      site: <int>
				      rack: <int>
				      encl: <int>
				      ctrl: <int>
				      disk: <int>

				# Profile is a reference to pools.  When Motr client is started, it receives
				# profile fid from the command line.  Motr client can only use the pools
				# referred to by its profile.
				#
				profiles:  # This section is optional.  If it is missing, a single "default"
				           # profile referring to all pools will be created.
				  - name: <str>
				    pools: [ <str> ]

				# FDMI filter defines if an FDMI record would be sent from FDMI source to FDMI
				# plugin. Only M0_FDMI_FILTER_TYPE_KV_SUBSTRING filters are supported.
				#
				fdmi_filters:  # This section is optional.  If it is missing, no fdmi filters
				               # would be defined
				  - name: <str>  # Human-readable name of the filter
				    node: <str>  # Hostname of the node that hosts FDMI plugin
				                 # that would receive every FDMI record this filter matches
				    client_index: <int>  # 0-based index of the client that would be
				                         # the FDMI plugin. Only "other" clients could be FDMI
				                         # plugins. "s3" clients are not used to for this
				                         # index.
				                         # The client has to be defined in m0_clients section
				                         # already.
				    substrings: [ <str> ]  # List of strings for FDMI plugin to match
				                           # See m0_conf_fdmi_filter::ff_substrings
				                           # for the reference. May be empty or absent.
				...  # end of the document (optional)
			# MY VER: see Word
				# SEARCH NETWORK INTERFACE
					ip a
					# Choose one that appear after executing this command
				# CREATE DEVICES SPECIFIED IN IO_DISKS
					sudo mkdir -p /var/motr
					for i in {0..9}; do
					    sudo dd if=/dev/zero of=/var/motr/disk$i.img bs=1M seek=9999 count=1
					    sudo losetup /dev/loop$i /var/motr/disk$i.img
					done
				cp /opt/seagate/cortx/hare/share/cfgen/examples/singlenode.yaml ~/CDF.yaml
				vi ~/CDF.yaml
				# Note on VIM EDITOR
					# press esc to quit insert mode
					# then type : followed by one of:
					# cq -> quit without saving
					# wq -> write and quit
					# etc.
				hctl bootstrap --mkfs ~/CDF.yaml

				# ERROR


# CORTX ON KUBERNETES
- Deploy CORTX inside Kubernetes: https://github.com/Seagate/cortx-re/blob/main/solutions/community-deploy/CORTX-Deployment.md

# COMPILE AND RUN MOTR ONLY

1. Setup & Build
	eval "$(ssh-agent -s)"
	ssh cc@129.114.109.241
	sudo passwd -d root
	su -
	git clone --recursive https://github.com/Seagate/cortx-motr.git
	cd cortx-motr && sudo ./scripts/install-build-deps

	# Error --> CentOS Linux 8 - AppStream Error: Failed to download metadata for repo 'appstream'
		cd /etc/yum.repos.d/
		sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
		sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
	
	cd cortx-motr && sudo ./scripts/install-build-deps

	# Error --> Error: Problem: cannot install the best candidate for the job - nothing provides (ansible-core >= 2.12.2 with ansible-core < 2.13) needed by ansible-5.4.0-2.el8.noarch
	
	# Turned out that CentOS8 is no longer maintained (EOL). I didn't continue using them since ansible is hardly installed
