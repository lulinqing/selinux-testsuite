
POLDEV ?= /usr/share/selinux/devel
SEMODULE = /usr/sbin/semodule
CHECKPOLICY = /usr/bin/checkpolicy
CHECKMODULE = /usr/bin/checkmodule

DISTRO=$(shell ../tests/os_detect)
RHEL_VERS=$(shell echo $(DISTRO) | sed 's/RHEL//')

POL_VERS := $(shell $(CHECKPOLICY) -V |cut -f 1 -d ' ')
MOD_POL_VERS := $(shell $(CHECKMODULE) -V |cut -f 2 -d '-')
SELINUXFS := $(shell cat /proc/mounts | grep selinuxfs | cut -f 2 -d ' ')
MAX_KERNEL_POLICY := $(shell cat $(SELINUXFS)/policyvers)

TARGETS = \
	test_global.te test_capable_file.te test_capable_net.te \
	test_capable_sys.te test_dyntrace.te test_dyntrans.te \
	test_entrypoint.te test_execshare.te test_exectrace.te \
	test_execute_no_trans.te test_fdreceive.te test_file.te \
	test_inherit.te test_ioctl.te test_ipc.te test_link.te test_mkdir.te \
	test_nnp_nosuid.te test_open.te test_ptrace.te test_readlink.te \
	test_relabel.te test_rename.te test_rxdir.te test_setattr.te \
	test_setnice.te test_sigkill.te test_stat.te test_sysctl.te \
	test_task_create.te test_task_getpgid.te test_task_getsched.te \
	test_task_getsid.te test_task_setpgid.te test_task_setsched.te \
	test_transition.te test_inet_socket.te test_unix_socket.te \
	test_mmap.te test_overlayfs.te test_mqueue.te test_mac_admin.te \
	test_ibpkey.te test_atsecure.te

ifeq ($(shell [ $(POL_VERS) -ge 24 ] && echo true),true)
TARGETS += test_bounds.te
endif

ifeq ($(shell [ $(MOD_POL_VERS) -ge 18 -a $(MAX_KERNEL_POLICY) -ge 30 ] && echo true),true)
TARGETS += test_ioctl_xperms.te
endif

ifeq ($(shell grep -q cap_userns $(POLDEV)/include/support/all_perms.spt && echo true),true)
TARGETS += test_cap_userns.te
endif

ifeq ($(shell grep -q icmp_socket $(POLDEV)/include/support/all_perms.spt && echo true),true)
TARGETS += test_extended_socket_class.te
endif

ifeq ($(shell grep -q netlink_iscsi_socket $(POLDEV)/include/support/all_perms.spt && echo true),true)
TARGETS += test_netlink_socket.te
endif

ifeq ($(shell grep -q getrlimit $(POLDEV)/include/support/all_perms.spt && echo true),true)
TARGETS += test_prlimit.te
endif

ifeq ($(shell grep -q infiniband_endport $(POLDEV)/include/support/all_perms.spt && echo true),true)
TARGETS += test_ibendport.te
endif

ifeq ($(shell grep -q all_file_perms.*map $(POLDEV)/include/support/all_perms.spt && echo true),true)
export M4PARAM = -Dmap_permission_defined
endif

ifeq ($(shell grep -q nnp_transition $(POLDEV)/include/support/all_perms.spt && echo true),true)
export M4PARAM += -Dnnp_nosuid_transition_permission_defined
endif

ifeq ($(shell grep -q binder $(POLDEV)/include/support/all_perms.spt && echo true),true)
TARGETS += test_binder.te
endif

ifeq ($(shell grep -q corenet_sctp_bind_all_nodes $(POLDEV)/include/kernel/corenetwork.if && echo true),true)
TARGETS += test_sctp.te
endif

ifeq (x$(DISTRO),$(filter x$(DISTRO),xRHEL4 xRHEL5 xRHEL6))
TARGETS:=$(filter-out test_overlayfs.te test_mqueue.te, $(TARGETS))
endif

ifeq (x$(DISTRO),$(filter x$(DISTRO), xRHEL6))
TARGETS:=$(filter-out test_ibpkey.te, $(TARGETS))
endif

ifeq (x$(DISTRO),$(filter x$(DISTRO),xRHEL4 xRHEL5))
	BUILD_TARGET := build_rhel
	LOAD_TARGET := load_rhel
	UNLOAD_TARGET := unload_rhel
else
	BUILD_TARGET := build_general
	LOAD_TARGET := load_general
	UNLOAD_TARGET := unload_general
endif

all: build

build: $(BUILD_TARGET)
load: $(LOAD_TARGET)
unload: $(UNLOAD_TARGET)

expand_check:
	# Test for "expand-check = 0" in /etc/selinux/semanage.conf
	@grep -q '^[ \t]*expand-check[ \t]*=[ \t]*0' /etc/selinux/semanage.conf || \
		(echo "ERROR: set 'expand-check = 0' in /etc/selinux/semanage.conf"; \
		 /bin/false)

build_rhel: $(TARGETS)
	# RHEL specific policy build
	$(MAKE) -C redhat/$(RHEL_VERS) all

build_general: $(TARGETS)
	# General policy build
	@if [ -d $(POLDEV) ]; then \
		mkdir -p test_policy; \
		cp test_policy.if test_policy; \
		set -e; rm -f test_policy.te; \
		cat $(TARGETS) > test_policy/test_policy.te; \
		$(MAKE) -C test_policy -f $(POLDEV)/Makefile test_policy.pp; \
	else \
		echo "ERROR: You must have selinux-policy-devel installed."; \
	fi

load_rhel: expand_check all
	# RHEL specific policy load
	$(MAKE) -C redhat/$(RHEL_VERS) load

load_general: expand_check all
	# General policy load
	@-/usr/sbin/setsebool allow_domain_fd_use=0
	$(SEMODULE) -i test_policy/test_policy.pp

unload_rhel:
	# RHEL specific policy unload
	$(MAKE) -C redhat/$(RHEL_VERS) unload

unload_general:
	# General policy unload
	@-/usr/sbin/setsebool allow_domain_fd_use=1
	$(SEMODULE) -r test_policy

clean:
	rm -rf test_policy tmp
