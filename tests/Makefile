export TESTSROOTDIR:=$(shell pwd)
SRCHOME:=$(shell readlink -f $(TESTSROOTDIR)/../ || realpath $(TESTSROOTDIR)/../ )
ifeq ($(SRCHOME),)
  SRCHOME:=$(TESTSROOTDIR)/..
endif
export SRCHOME
ifeq ($(TESTID),)
export TESTID:=$(shell $(TESTSROOTDIR)/tools/get_random.sh)
endif
include $(TESTSROOTDIR)/Makefile.common

# List of disabled tests
DISABLED_TESTS:=$(shell awk '/^DISABLED/{f=1;next};/<END>/{f=0}f' TODO | cut -d' ' -f1)

ifeq ($(SKIPSSL), 1)
SKIPS:=$(shell grep ssl *.test/lrl*  | grep 'ssl_client_mode VERIFY' | cut -f1 -d'/' | sort -u)
DISABLED_TESTS:="${DISABLED_TESTS} ${SKIPS}"
endif

ifeq ($(wildcard $(BUILDDIR)/tcl/libtclcdb2.so),)
#if we have not compiled with tcl, skip tests that require tclcdb2
SKIPS:=$(shell ls *.test/*.tcl | xargs grep 'package require tclcdb2' | cut -f1 -d'/' | sort -u)
DISABLED_TESTS:="${DISABLED_TESTS} ${SKIPS}"
endif

export DISABLED_TESTS

DOALL:=0

ifeq ($(subst all,,$(MAKECMDGOALS)),) #if cmdgoals empty or 'all'
DOALL:=1
endif

# List of all tests that need to run. This is essentially a list of all
# non-disabled tests.
export ALL_TESTS:=$(patsubst %.test, %, $(filter-out $(DISABLED_TESTS), \
    $(shell $(TESTSROOTDIR)/tools/get_tests_inorder.sh)))

TOTAL=$(words $(MAKECMDGOALS) )
ifeq ($(DOALL),1)
TOTAL=$(words $(ALL_TESTS) )
endif

all: init basicops all_tests

init:
	@mkdir -p ${TESTDIR}

# Run all non-disabled tests.
all_tests: $(ALL_TESTS)

basicops_nokey: init
	$(shell which git > /dev/null 2>&1 && git rev-parse --abbrev-ref HEAD | xargs echo "Branch" >> ${TESTDIR}/test.log)
	$(shell which git > /dev/null 2>&1 && git rev-parse --short HEAD | xargs echo "SHA " >> ${TESTDIR}/test.log)
	$(shell echo "TESTID=${TESTID} " >> ${TESTDIR}/test.log)
	$(shell echo "CLUSTER=${CLUSTER} " >> ${TESTDIR}/test.log)
	$(shell echo "SRCHOME=${SRCHOME} " >> ${TESTDIR}/test.log)
	$(shell echo "TESTSROOTDIR=${TESTSROOTDIR} " >> ${TESTDIR}/test.log)
	$(shell echo "TESTDIR=${TESTDIR} " >> ${TESTDIR}/test.log)
	$(shell echo "DISABLED= $(sort ${DISABLED_TESTS}) " >> ${TESTDIR}/test.log)
	$(shell echo "MAKECMDGOALS=${MAKECMDGOALS}" >> ${TESTDIR}/test.log)
	$(shell echo "" >> ${TESTDIR}/test.log)

basicops: basicops_nokey
	$(shell TESTDIR="${TESTDIR}" CLUSTER="${CLUSTER}" SKIPSSL="${SKIPSSL}" $(TESTSROOTDIR)/tools/keygen.sh)
	$(shell TOTAL="${TOTAL}" $(TESTSROOTDIR)/tools/smalldelay.sh)
	$(shell TESTDIR="${TESTDIR}" CLUSTER="${CLUSTER}" TESTSROOTDIR="${TESTSROOTDIR}" COMDB2_EXE=${COMDB2_EXE} CDB2SQL_EXE=${CDB2SQL_EXE} COMDB2AR_EXE=${COMDB2AR_EXE} PMUX_EXE=${PMUX_EXE} PMUXPORT=${PMUXPORT} SKIP_COPY_EXE="${SKIP_COPY_EXE}" ${TESTSROOTDIR}/tools/copy_files_to_cluster.sh > ${TESTDIR}/copy_files_to_cluster.log 2>&1 || echo "exit 1 copy_files_to_cluster failed, see ${TESTDIR}/copy_files_to_cluster.log" )

showparams: 
	@echo MAKECMDGOALS=${MAKECMDGOALS}
	@echo DOALL=${DOALL}
	@echo TESTSROOTDIR=${TESTSROOTDIR}
	@echo SRCHOME=${SRCHOME}
	@echo TESTID=${TESTID}
	@echo TESTDIR=${TESTDIR}
	@echo CLUSTER=${CLUSTER}
	@echo COMDB2AR_EXE=${COMDB2AR_EXE}
	@echo COMDB2_EXE=${COMDB2_EXE}
	@echo CDB2SQL_EXE=${CDB2SQL_EXE}
	@echo COPYCOMDB2_EXE=${COPYCOMDB2_EXE}
	@echo CDB2_SQLREPLAY_EXE=${CDB2_SQLREPLAY_EXE}
	@echo PMUX_EXE=${PMUX_EXE}



%_generated: basicops
	@N=`$(TESTSROOTDIR)/tools/get_test_counter.sh` && echo TESTID=${TESTID} running in $(TESTDIR) $@ $$N/${TOTAL}
	@tools/create_generated_tests.sh `tools/get_testopts_path.sh $@` $(TESTDIR)
	@$(MAKE) -skC $(TESTDIR)/$@.test

$(patsubst %.test,%,$(ALL_TESTS)): basicops

%: %.test
	@N=`$(TESTSROOTDIR)/tools/get_test_counter.sh` && echo TESTID=${TESTID} running in $(TESTDIR) $* $$N/${TOTAL}
	@cp -r $< $(TESTDIR)/
	@$(MAKE) -skC $(TESTDIR)/$<

%.clean: %.test
	+$(MAKE) -C $(patsubst %.clean,%,$<) clean

%.setup: %.test
	+$(MAKE) -C $(patsubst %.setup,%,$<) setup

%.tool: %.test
	+$(MAKE) -C $(patsubst %.tool,%,$<) tool

generated_tests:
	$(shell $(TESTSROOTDIR)/tools/create_generated_tests.sh)

stop:
	@./stopall

kill:
	@./dokillall

testtotalclean: $(patsubst %.test,%.clean,$(shell ls -d *.test))
	rm -f logs/*
	rm -f *.test/*.res
	rm -f *.test/comdb2db.cfg
	rm -f .*.failed
	rm -fr dbs
	rm -f $(TMPDIR)/cdb2test*

clean: testclean

testclean:
	@rm -rf test_[0-9]* $(TESTDIR)/ *_generated.test/
	@$(foreach n, $(CLUSTER), ssh $(n) 'rm -rf $(TESTSROOTDIR)/test_[0-9]* $(TESTDIR)/' < /dev/null;)

