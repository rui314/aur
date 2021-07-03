CC = clang
CXX = clang++

GIT_HASH ?= $(shell [ -d .git ] && git rev-parse HEAD)

CPPFLAGS = -g -Imimalloc/include -pthread -std=c++20 \
           -DMOLD_VERSION=\"0.9.1\" \
           -DGIT_HASH=\"$(GIT_HASH)\" \
	   $(EXTRA_CPPFLAGS)
LDFLAGS += $(EXTRA_LDFLAGS)
LIBS = -Wl,-as-needed -lcrypto -pthread -lz -lxxhash -ldl
OBJS = main.o object_file.o input_sections.o output_chunks.o \
       mapfile.o perf.o linker_script.o archive_file.o output_file.o \
       subprocess.o gc_sections.o icf.o symbols.o cmdline.o filepath.o \
       passes.o tar.o compress.o memory_mapped_file.o relocatable.o \
       arch_x86_64.o arch_i386.o

PREFIX ?= /usr
DEBUG ?= 0
LTO ?= 0
ASAN ?= 0
TSAN ?= 0

ifeq ($(DEBUG), 1)
  CPPFLAGS += -O0
else
  CPPFLAGS += -O2
endif

ifeq ($(LTO), 1)
  CPPFLAGS += -flto -O3
  LDFLAGS  += -flto
endif

ifeq ($(ASAN), 1)
  CPPFLAGS += -fsanitize=address
  LDFLAGS  += -fsanitize=address
else
  # By default, we want to use mimalloc as a memory allocator.
  # Since replacing the standard malloc is not compatible with ASAN,
  # we do that only when ASAN is not enabled.
  ifdef SYSTEM_MIMALLOC
    LIBS += -lmimalloc
  else
    MIMALLOC_LIB = mimalloc/out/release/libmimalloc.a
    LIBS += -Wl,-whole-archive $(MIMALLOC_LIB) -Wl,-no-whole-archive
  endif
endif

ifeq ($(TSAN), 1)
  CPPFLAGS += -fsanitize=thread
  LDFLAGS  += -fsanitize=thread
endif

ifdef SYSTEM_TBB
  LIBS += -ltbb
else
  TBB_LIB = oneTBB/out/libs/libtbb.a
  LIBS += $(TBB_LIB)
  CPPFLAGS += -IoneTBB/include
endif

all: mold mold-wrapper.so

mold: $(OBJS) $(MIMALLOC_LIB) $(TBB_LIB)
	$(CXX) $(CXXFLAGS) $(OBJS) -o $@ $(LDFLAGS) $(LIBS)

mold-wrapper.so: mold-wrapper.c Makefile
	$(CC) -fPIC -shared -o $@ $< -ldl

$(OBJS): mold.h elf.h Makefile

$(MIMALLOC_LIB):
	mkdir -p mimalloc/out/release
	(cd mimalloc/out/release; CFLAGS=-DMI_USE_ENVIRON=0 cmake ../..)
	$(MAKE) -C mimalloc/out/release mimalloc-static

$(TBB_LIB):
	mkdir -p oneTBB/out
	(cd oneTBB/out; cmake -DBUILD_SHARED_LIBS=OFF -DTBB_TEST=OFF -DCMAKE_CXX_FLAGS=-D__TBB_DYNAMIC_LOAD_ENABLED=0 ..)
	$(MAKE) -C oneTBB/out tbb
	(cd oneTBB/out; ln -sf *_relwithdebinfo libs)

test tests check: all
	 $(MAKE) -C test --output-sync --no-print-directory

install: all
	install -m 755 -d $(DESTDIR)$(PREFIX)/bin
	install -m 755 mold $(DESTDIR)$(PREFIX)/bin
	strip $(DESTDIR)$(PREFIX)/bin/mold

	install -m 755 -d $(DESTDIR)$(PREFIX)/lib/mold
	install -m 644 mold-wrapper.so $(DESTDIR)$(PREFIX)/lib/mold
	strip $(DESTDIR)$(PREFIX)/lib/mold/mold-wrapper.so

	install -m 755 -d $(DESTDIR)$(PREFIX)/share/man/man1
	install -m 644 docs/mold.1 $(DESTDIR)$(PREFIX)/share/man/man1
	rm -f $(DESTDIR)$(PREFIX)/share/man/man1/mold.1.gz
	gzip -9 $(DESTDIR)$(PREFIX)/share/man/man1/mold.1

uninstall:
	rm -f $(DESTDIR)$(PREFIX)/bin/mold
	rm -f $(DESTDIR)$(PREFIX)/share/man/man1/mold.1.gz
	rm -rf $(DESTDIR)$(PREFIX)/lib/mold

clean:
	rm -rf *.o *~ mold mold-wrapper.so test/tmp mimalloc/out oneTBB/out

.PHONY: all test tests check clean $(TESTS)
