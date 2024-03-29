# SPDX-License-Identifier: GPL-2.0

LIBDIR := ../../../lib
BPFDIR := $(LIBDIR)/bpf
APIDIR := ../../../include/uapi
GENDIR := ../../../../include/generated
GENHDR := $(GENDIR)/autoconf.h

ifneq ($(wildcard $(GENHDR)),)
  GENFLAGS := -DHAVE_GENHDR
endif

CLANG		?= clang
LLC		?= llc
LLVM_OBJCOPY	?= llvm-objcopy
LLVM_READELF	?= llvm-readelf
BTF_PAHOLE	?= pahole
CFLAGS += -Wall -O2 -I$(APIDIR) -I$(LIBDIR) -I$(BPFDIR) -I$(GENDIR) $(GENFLAGS) -I../../../include
LDLIBS += -lcap -lelf -lrt -lpthread

# Order correspond to 'make run_tests' order
TEST_GEN_PROGS = test_verifier test_tag test_maps test_lru_map test_lpm_map test_progs \
	test_align test_verifier_log test_dev_cgroup test_tcpbpf_user \
	test_sock test_btf test_sockmap test_lirc_mode2_user get_cgroup_id_user \
	test_socket_cookie test_cgroup_storage test_select_reuseport test_section_names \
	test_netcnt test_tcpnotify_user

BPF_OBJ_FILES = \
	test_xdp_redirect.o test_xdp_meta.o sockmap_parse_prog.o \
	sockmap_verdict_prog.o dev_cgroup.o sample_ret0.o \
	test_tcpnotify_kern.o sample_map_ret0.o test_tcpbpf_kern.o \
	sockmap_tcp_msg_prog.o connect4_prog.o connect6_prog.o \
	test_btf_haskv.o test_btf_nokv.o test_sockmap_kern.o \
	test_tunnel_kern.o test_sockhash_kern.o test_lwt_seg6local.o \
	sendmsg4_prog.o sendmsg6_prog.o test_lirc_mode2_kern.o \
	get_cgroup_id_kern.o socket_cookie_prog.o test_select_reuseport_kern.o \
	test_skb_cgroup_id_kern.o bpf_flow.o netcnt_prog.o test_xdp_vlan.o \
	xdp_dummy.o test_map_in_map.o

# Objects are built with default compilation flags and with sub-register
# code-gen enabled.
BPF_OBJ_FILES_DUAL_COMPILE = \
	test_pkt_access.o test_pkt_access.o test_xdp.o test_adjust_tail.o \
	test_l4lb.o test_l4lb_noinline.o test_xdp_noinline.o test_tcp_estats.o \
	test_obj_id.o test_pkt_md_access.o test_tracepoint.o \
	test_stacktrace_map.o test_stacktrace_map.o test_stacktrace_build_id.o \
	test_stacktrace_build_id.o test_get_stack_rawtp.o \
	test_get_stack_rawtp.o test_tracepoint.o test_sk_lookup_kern.o \
	test_queue_map.o test_stack_map.o

TEST_GEN_FILES = $(BPF_OBJ_FILES) $(BPF_OBJ_FILES_DUAL_COMPILE)

# Also test sub-register code-gen if LLVM + kernel both has eBPF v3 processor
# support which is the first version to contain both ALU32 and JMP32
# instructions.
SUBREG_CODEGEN := $(shell echo "int cal(int a) { return a > 0; }" | \
			$(CLANG) -target bpf -O2 -emit-llvm -S -x c - -o - | \
			$(LLC) -mattr=+alu32 -mcpu=probe 2>&1 | \
			grep 'if w')
ifneq ($(SUBREG_CODEGEN),)
TEST_GEN_FILES += $(patsubst %.o,alu32/%.o, $(BPF_OBJ_FILES_DUAL_COMPILE))
endif

# Order correspond to 'make run_tests' order
TEST_PROGS := test_kmod.sh \
	test_libbpf.sh \
	test_xdp_redirect.sh \
	test_xdp_meta.sh \
	test_offload.py \
	test_sock_addr.sh \
	test_tunnel.sh \
	test_lwt_seg6local.sh \
	test_lirc_mode2.sh \
	test_skb_cgroup_id.sh \
	test_flow_dissector.sh \
	test_xdp_vlan.sh

TEST_PROGS_EXTENDED := with_addr.sh \
	with_tunnels.sh \
	tcp_client.py \
	tcp_server.py

# Compile but not part of 'make run_tests'
TEST_GEN_PROGS_EXTENDED = test_libbpf_open test_sock_addr test_skb_cgroup_id_user \
	flow_dissector_load test_flow_dissector

include ../lib.mk

# NOTE: $(OUTPUT) won't get default value if used before lib.mk
TEST_CUSTOM_PROGS = $(OUTPUT)/urandom_read
all: $(TEST_CUSTOM_PROGS)

$(OUTPUT)/urandom_read: $(OUTPUT)/%: %.c
	$(CC) -o $@ -static $< -Wl,--build-id

BPFOBJ := $(OUTPUT)/libbpf.a

$(TEST_GEN_PROGS): $(BPFOBJ)

$(TEST_GEN_PROGS_EXTENDED): $(OUTPUT)/libbpf.a

$(OUTPUT)/test_dev_cgroup: cgroup_helpers.c
$(OUTPUT)/test_skb_cgroup_id_user: cgroup_helpers.c
$(OUTPUT)/test_sock: cgroup_helpers.c
$(OUTPUT)/test_sock_addr: cgroup_helpers.c
$(OUTPUT)/test_socket_cookie: cgroup_helpers.c
$(OUTPUT)/test_sockmap: cgroup_helpers.c
$(OUTPUT)/test_tcpbpf_user: cgroup_helpers.c
$(OUTPUT)/test_tcpnotify_user: cgroup_helpers.c trace_helpers.c
$(OUTPUT)/test_progs: trace_helpers.c
$(OUTPUT)/get_cgroup_id_user: cgroup_helpers.c
$(OUTPUT)/test_cgroup_storage: cgroup_helpers.c
$(OUTPUT)/test_netcnt: cgroup_helpers.c

.PHONY: force

# force a rebuild of BPFOBJ when its dependencies are updated
force:

$(BPFOBJ): force
	$(MAKE) -C $(BPFDIR) OUTPUT=$(OUTPUT)/

PROBE := $(shell $(LLC) -march=bpf -mcpu=probe -filetype=null /dev/null 2>&1)

# Let newer LLVM versions transparently probe the kernel for availability
# of full BPF instruction set.
ifeq ($(PROBE),)
  CPU ?= probe
else
  CPU ?= generic
endif

# Get Clang's default includes on this system, as opposed to those seen by
# '-target bpf'. This fixes "missing" files on some architectures/distros,
# such as asm/byteorder.h, asm/socket.h, asm/sockios.h, sys/cdefs.h etc.
#
# Use '-idirafter': Don't interfere with include mechanics except where the
# build would have failed anyways.
CLANG_SYS_INCLUDES := $(shell $(CLANG) -v -E - </dev/null 2>&1 \
	| sed -n '/<...> search starts here:/,/End of search list./{ s| \(/.*\)|-idirafter \1|p }')

CLANG_FLAGS = -I. -I./include/uapi -I../../../include/uapi \
	      $(CLANG_SYS_INCLUDES) \
	      -Wno-compare-distinct-pointer-types

$(OUTPUT)/test_l4lb_noinline.o: CLANG_FLAGS += -fno-inline
$(OUTPUT)/test_xdp_noinline.o: CLANG_FLAGS += -fno-inline

$(OUTPUT)/test_queue_map.o: test_queue_stack_map.h
$(OUTPUT)/test_stack_map.o: test_queue_stack_map.h

$(OUTPUT)/flow_dissector_load.o: flow_dissector_load.h
$(OUTPUT)/test_progs.o: flow_dissector_load.h

BTF_LLC_PROBE := $(shell $(LLC) -march=bpf -mattr=help 2>&1 | grep dwarfris)
BTF_PAHOLE_PROBE := $(shell $(BTF_PAHOLE) --help 2>&1 | grep BTF)
BTF_OBJCOPY_PROBE := $(shell $(LLVM_OBJCOPY) --help 2>&1 | grep -i 'usage.*llvm')
BTF_LLVM_PROBE := $(shell echo "int main() { return 0; }" | \
			  $(CLANG) -target bpf -O2 -g -c -x c - -o ./llvm_btf_verify.o; \
			  $(LLVM_READELF) -S ./llvm_btf_verify.o | grep BTF; \
			  /bin/rm -f ./llvm_btf_verify.o)

ifneq ($(BTF_LLVM_PROBE),)
	CLANG_FLAGS += -g
else
ifneq ($(BTF_LLC_PROBE),)
ifneq ($(BTF_PAHOLE_PROBE),)
ifneq ($(BTF_OBJCOPY_PROBE),)
	CLANG_FLAGS += -g
	LLC_FLAGS += -mattr=dwarfris
	DWARF2BTF = y
endif
endif
endif
endif

ifneq ($(SUBREG_CODEGEN),)
ALU32_BUILD_DIR = $(OUTPUT)/alu32
TEST_CUSTOM_PROGS += $(ALU32_BUILD_DIR)/test_progs_32
$(ALU32_BUILD_DIR):
	mkdir -p $@

$(ALU32_BUILD_DIR)/urandom_read: $(OUTPUT)/urandom_read
	cp $< $@

$(ALU32_BUILD_DIR)/test_progs_32: test_progs.c $(ALU32_BUILD_DIR) \
						$(ALU32_BUILD_DIR)/urandom_read
	$(CC) $(CFLAGS) -o $(ALU32_BUILD_DIR)/test_progs_32 $< \
		trace_helpers.c $(OUTPUT)/libbpf.a $(LDLIBS)

$(ALU32_BUILD_DIR)/%.o: %.c $(ALU32_BUILD_DIR) $(ALU32_BUILD_DIR)/test_progs_32
	$(CLANG) $(CLANG_FLAGS) \
		 -O2 -target bpf -emit-llvm -c $< -o - |      \
	$(LLC) -march=bpf -mattr=+alu32 -mcpu=$(CPU) $(LLC_FLAGS) \
		-filetype=obj -o $@
ifeq ($(DWARF2BTF),y)
	$(BTF_PAHOLE) -J $@
endif
endif

# Have one program compiled without "-target bpf" to test whether libbpf loads
# it successfully
$(OUTPUT)/test_xdp.o: test_xdp.c
	$(CLANG) $(CLANG_FLAGS) \
		-O2 -emit-llvm -c $< -o - | \
	$(LLC) -march=bpf -mcpu=$(CPU) $(LLC_FLAGS) -filetype=obj -o $@
ifeq ($(DWARF2BTF),y)
	$(BTF_PAHOLE) -J $@
endif

$(OUTPUT)/%.o: %.c
	$(CLANG) $(CLANG_FLAGS) \
		 -O2 -target bpf -emit-llvm -c $< -o - |      \
	$(LLC) -march=bpf -mcpu=$(CPU) $(LLC_FLAGS) -filetype=obj -o $@
ifeq ($(DWARF2BTF),y)
	$(BTF_PAHOLE) -J $@
endif

$(OUTPUT)/test_verifier: $(OUTPUT)/verifier/tests.h
$(OUTPUT)/test_verifier: CFLAGS += -I$(OUTPUT)

VERIFIER_TEST_FILES := $(wildcard verifier/*.c)
$(OUTPUT)/verifier/tests.h: $(VERIFIER_TEST_FILES)
	$(shell ( cd verifier/
		  echo '/* Generated header, do not edit */'; \
		  echo '#ifdef FILL_ARRAY'; \
		  ls *.c 2> /dev/null | \
			sed -e 's@\(.*\)@#include \"\1\"@'; \
		  echo '#endif' \
		 ) > $(OUTPUT)/verifier/tests.h)

EXTRA_CLEAN := $(TEST_CUSTOM_PROGS) $(ALU32_BUILD_DIR)
