################################################################################
#                            Variables to override                             #
################################################################################

EXTRA_INCLUDE_PATHS=
EXTRA_LIB_PATHS=
EXTRA_CC_FLAGS=
BYTECODE=

################################################################################
#                              OS-dependent stuff                              #
################################################################################

OS=$(shell uname -s)
ROOT=$(shell pwd)

ifeq ($(OS), Linux)
  INOTIFY=third-party/inotify
  FSNOTIFY=fsnotify_linux
  ELF=elf
  RT=rt
  FRAMEWORKS=
  SECTCREATE=
endif
ifeq ($(OS), Darwin)
  INOTIFY=fsevents
  FSNOTIFY=fsnotify_darwin
  ELF=
  RT=
  FRAMEWORKS=CoreServices CoreFoundation
  SECTCREATE=-cclib -sectcreate -cclib __text -cclib hhi -cclib $(ROOT)/../bin/hhi.tar.gz
endif

ifeq ($(BYTECODE), 1)
  TARGET_EXT=byte
else
  TARGET_EXT=native
endif

################################################################################
#                                 Definitions                                  #
################################################################################

MODULES=\
  ast\
  client\
  decl\
  deps\
  dfind\
  diff\
  emitter\
  debug\
  find\
  format\
	full_fidelity\
	full_fidelity/schema\
  hackfmt/debug\
  hackfmt/line_splitter\
  hackfmt\
  globals\
  h2tp/common\
  h2tp/mapper\
  h2tp/unparser\
  h2tp\
  hackfmt/error\
  heap\
  hh_matcher\
  hh_matcher/test\
  hhbc\
  hhi\
  ide_rpc\
  injection/default_injector\
  libancillary\
  naming\
  options\
  parsing\
  procs\
  recorder\
  search\
  server\
  socket\
  stubs\
  third-party/avl\
  third-party/core\
  typing\
  utils\
  utils/collections\
  utils/disk\
  utils/process\
  utils/hg\
  utils/hh_json\
  monitor\
  watchman\
  $(FSNOTIFY)\
  $(INOTIFY)

NATIVE_C_FILES=\
  heap/hh_shared.c\
  hhi/hhi_elf.c\
  hhi/hhi_win32res_stubs.c\
  libancillary/libancillary-stubs.c\
  third-party/libancillary/fd_recv.c\
  third-party/libancillary/fd_send.c\
  utils/files.c\
  utils/get_build_id.c\
  utils/handle_stubs.c\
  utils/nproc.c\
  utils/priorities.c\
  utils/realpath.c\
  utils/sysinfo.c\
  utils/win32_support.c\
  $(INOTIFY)/$(notdir $(INOTIFY))_stubs.c

OCAML_LIBRARIES=\
  str\
  unix\
  bigarray

NATIVE_LIBRARIES=\
  pthread\
  $(ELF)\
  $(RT)

TARGETS_BASE=_build/hh_server _build/hh_client \
	_build/hh_single_type_check _build/hh_emitter \
	_build/hh_format _build/h2tp \
	_build/hackfmt \
	_build/hh_match \
	_build/code_extent_tests \
	_build/matcher_test \
	_build/patcher_api_test \
	_build/patcher_module_test \
	_build/patcher_test

TARGETS=$(foreach target,$(TARGETS_BASE),$(target).$(TARGET_EXT))

# Find all source files (all files not in _build dir)
ALL_SRC_FILES=$(call rwildcard,$(patsubst _build,,$(wildcard *)),*.*)

################################################################################
#                                    Rules                                     #
################################################################################

OBJECT_FILES_TO_BUILD=$(patsubst %.c,%.o,$(NATIVE_C_FILES))
NATIVE_OBJECT_FILES=$(OBJECT_FILES_TO_BUILD) utils/get_build_id.gen.o
INCLUDE_OPTS=$(foreach dir,$(MODULES),-I $(dir))
LIB_OPTS=$(foreach lib,$(OCAML_LIBRARIES),-lib $(lib))
NATIVE_LIB_OPTS=$(foreach lib, $(NATIVE_LIBRARIES),-cclib -l$(lib))
EXTRA_NATIVE_LIB_OPTS=$(foreach lib, $(EXTRA_NATIVE_LIBRARIES),-cclib -l$(lib))
EXTRA_INCLUDE_OPTS=$(foreach dir, $(EXTRA_INCLUDE_PATHS),-ccopt -I$(dir))
EXTRA_CC_OPTS=$(foreach opt, $(EXTRA_CC_FLAGS),-ccopt $(opt))
EXTRA_LIB_OPTS=$(foreach dir, $(EXTRA_LIB_PATHS),-cclib -L$(dir))
FRAMEWORK_OPTS=$(foreach framework, $(FRAMEWORKS),-cclib -framework -cclib $(framework))

LINKER_FLAGS=$(NATIVE_OBJECT_FILES) $(NATIVE_LIB_OPTS) $(EXTRA_LIB_OPTS) \
	     $(EXTRA_NATIVE_LIB_OPTS) $(FRAMEWORK_OPTS) $(SECTCREATE)
# Function to recursively find files, eg: $(call rwildcard,dir/to/look/in/,*.c)
rwildcard=$(wildcard $1$2) $(foreach d,$(wildcard $1*),$(call rwildcard,$d/,$2))

all: build-hhi-archive build-hack copy-hack-files
all-ocp: build-hhi-archive build-hack-with-ocp copy-hack-files-ocp

clean:
	$(OCAMLBUILD) -clean
	find ../bin -mindepth 1 -not -path ../bin/README -delete
	rm -f utils/get_build_id.gen.c

build-hack: $(TARGETS) build-hhi-archive

build-hack-with-ocp: build-hhi-archive _build/utils/get_build_id.gen.o
	[ -d ${ROOT}/../_obuild ] || ( cd ${ROOT}/.. && ocp-build init )
	ocp-build build

# These are needed so we can turn OCAMLBUILD from 'a,b,c' into 'a b c' below
comma := ,
space :=
space +=

# All targets are built in one time, so no parallelization is necessary
.NOTPARALLEL: $(TARGETS)
# As there is no efficient way to calculate the dependencies of
# the targets, we make them dependent on all files. In doing this
# we ensure that no rebuild is necessary if nothing has changed
$(TARGETS): $(ALL_SRC_FILES)
	# build-hack-native-deps is a dependency of $(TARGETS) but as it is phony
	# we place it here to avoid unnecessary rebuilds
	$(MAKE) build-hack-native-deps build-hhi-archive
	$(OCAMLBUILD) $(subst $(comma),$(space),$(OCAMLBUILD_FLAGS)) -no-links -cflag -g $(INCLUDE_OPTS) $(LIB_OPTS) \
		-lflags "-g $(LINKER_FLAGS)" -tag custom \
		$(patsubst _build/%,%,$(TARGETS))
	# Touching the targets is necessary because the ocaml build
	# doesn't change the modification dates of the targets if
	# the new binaries are exactly the same as the old ones
	touch $(TARGETS)

build-hack-native-deps: $(NATIVE_C_FILES) _build/utils/get_build_id.gen.o
	$(OCAMLBUILD) $(subst $(comma),$(space),$(OCAMLBUILD_FLAGS)) -cflags "-g $(EXTRA_INCLUDE_OPTS) $(EXTRA_CC_OPTS)" $(OBJECT_FILES_TO_BUILD)

.PHONY: _build/utils/get_build_id.gen.c # run every time, depends on git commit
_build/utils/get_build_id.gen.c:
	mkdir -p $(dir $@)
	cd $(ROOT)/.. && \
        ocaml -I scripts -w -3 unix.cma scripts/gen_build_id.ml src/_build/utils/get_build_id.gen.c

_build/utils/get_build_id.gen.o: _build/utils/get_build_id.gen.c
	cd $(dir $@) && $(OCAMLC) $(EXTRA_INCLUDE_OPTS) $(EXTRA_CC_OPTS) -c $(notdir $<)

build-hhi-archive:
	mkdir -p ../bin
	$(MAKE) -C ../hhi

copy-hack-files: build-hack
	mkdir -p ../bin
ifeq ($(OS)$(BYTECODE), Linux)
	objcopy --add-section hhi=../bin/hhi.tar.gz _build/hh_server.$(TARGET_EXT) ../bin/hh_server
else
	cp _build/hh_server.$(TARGET_EXT) ../bin/hh_server
endif
	cp _build/hh_client.$(TARGET_EXT) ../bin/hh_client
	cp _build/hh_single_type_check.$(TARGET_EXT) ../bin/hh_single_type_check
	cp _build/hh_emitter.$(TARGET_EXT) ../bin/hh_emitter
	cp _build/hackfmt.$(TARGET_EXT) ../bin/hackfmt
	cp _build/hh_format.$(TARGET_EXT) ../bin/hh_format
	cp _build/h2tp/h2tp.$(TARGET_EXT) ../bin/h2tp
	cp _build/hh_match.$(TARGET_EXT) ../bin/hh_match

copy-hack-files-ocp: build-hack-with-ocp
	mkdir -p ../bin
ifeq ($(OS), Linux)
	objcopy --add-section hhi=../bin/hhi.tar.gz ../_obuild/hh_server/hh_server.asm ../bin/hh_server
else
	cp ../_obuild/hh_server/hh_server.asm ../bin/hh_server
endif
	cp ../_obuild/hh_client/hh_client.asm ../bin/hh_client
	cp ../_obuild/hh_single_type_check/hh_single_type_check.asm ../bin/hh_single_type_check
	cp ../_obuild/hh_format/hackfmt.asm ../bin/hackfmt
	cp ../_obuild/hh_format/hh_format.asm ../bin/hh_format
	cp ../_obuild/h2tp/h2tp.asm ../bin/h2tp
	cp ../_obuild/hh_match/hh_match.asm ../bin/hh_match

copy-match-test-files:
	mkdir -p ../bin
	cp _build/hh_matcher/test/code_extent_tests.$(TARGET_EXT) \
		../bin/code_extent_test_driver
	cp _build/hh_matcher/test/matcher_test.$(TARGET_EXT) \
		../bin/hh_match_test_driver
	cp _build/hh_matcher/test/patcher_module_test.$(TARGET_EXT) \
		../bin/hh_patch_module_test_driver
	cp _build/hh_matcher/test/patcher_api_test.$(TARGET_EXT) \
		../bin/hh_patch_api_test_driver
	cp _build/hh_matcher/test/patcher_test.$(TARGET_EXT) \
		../bin/hh_patch_test_driver

copy-match-test-files-ocp:
	mkdir -p ../bin
	cp ../_obuild/code_extent_tests/code_extent_tests.asm \
		../bin/code_extent_test_driver
	cp ../_obuild/matcher_test/matcher_test.asm \
		../bin/hh_match_test_driver
	cp ../_obuild/patcher_module_test/patcher_module_test.asm \
		../bin/hh_patch_module_test_driver
	cp ../_obuild/patcher_api_test/patcher_api_test.asm \
		../bin/hh_patch_api_test_driver
	cp ../_obuild/patcher_test/patcher_test.asm \
		../bin/hh_patch_test_driver

.PHONY: test test-ocp do-test
test: build-hack copy-hack-files copy-match-test-files
	${MAKE} do-test

test-ocp: build-hack-with-ocp copy-hack-files-ocp copy-match-test-files-ocp
	${MAKE} do-test

do-test:
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/autocomplete
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/color
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/colour
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/coverage
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/dumpsymbolinfo
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/dump_inheritance
	python3 ../test/verify.py --program ../bin/hh_format ../test/format
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/suggest
	python3 ../test/verify.py --program ../bin/hh_single_type_check ../test/typecheck
	python3 ../test/verify.py --program ../bin/hh_format ../test/typecheck \
		--disabled-extension .no_format \
		--out-extension .format_out \
		--expect-extension '' \
		--flags --root .
	python3 ../test/verify.py --program ../bin/code_extent_test_driver ../test/hh_matcher/code_extent
	python3 ../test/verify.py --program ../bin/hh_match_test_driver ../test/hh_matcher/matcher_tests
	python3 ../test/verify.py --program ../bin/hh_patch_module_test_driver ../test/hh_matcher/patcher_module_tests
	python3 ../test/verify.py --program ../bin/hh_patch_api_test_driver ../test/hh_matcher/patcher_api_tests
	python3 ../test/verify.py --program ../bin/hh_patch_test_driver ../test/hh_matcher/patcher_tests
	python3 ../test/integration/runner.py ../bin/hh_server ../bin/hh_client
