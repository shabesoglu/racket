
#################################################################
# This is the source for the `configure` script, to be compiled #
# by autoconf (use `make-configure` in "../racket").            #
#################################################################

AC_INIT([rktio.h])
AC_CONFIG_HEADERS([rktio_config.h])

AC_CONFIG_AUX_DIR(../lt)
AC_CANONICAL_SYSTEM

AC_ARG_ENABLE(shared,  [  --enable-shared         create shared libraries (ok, but not recommended)])
AC_ARG_ENABLE(pthread, [  --enable-pthread        link with pthreads (usually auto-enabled if needed)])
AC_ARG_ENABLE(iconv,   [  --enable-iconv          use iconv (usually auto-enabled)])

if test "${enable_iconv}" = "" ; then
  enable_iconv=yes
fi

###### Autoconfigure #######

AC_PROG_CC

# If using gcc, we want all warnings:
if test "$CC" = "gcc" ; then 
  CFLAGS="$CFLAGS -Wall"
fi

AC_PROG_RANLIB
if test "$AR" = '' ; then 
  AR="${ac_tool_prefix}ar"
  AC_CHECK_PROG(platform_ar_found, $AR, yes, no)
  if test "$platform_ar_found" = 'no' ; then
    AR="ar"
  fi
fi
if test "$ARFLAGS" = '' ; then 
  ARFLAGS=ruv
fi

AC_CHECK_LIB(m, fmod)
AC_CHECK_LIB(dl, dlopen)

############## platform tests ################

case "$host_os" in
  solaris2*)
    try_poll_syscall="no" # poll() has performance problems on Solaris?
    use_flag_pthread="no"
    use_flag_posix_pthread="yes"
    ;;
  aix*)
    ;;
  *freebsd*)
    enable_pthread_by_default=yes
    try_kqueue_syscall=yes
    ;;
  openbsd*)
    enable_pthread_by_default=yes
    try_kqueue_syscall=yes
    ;;
  bitrig*)
    enable_pthread_by_default=yes
    try_kqueue_syscall=yes
    ;;
  dragonfly*)
    enable_pthread_by_default=yes
    try_kqueue_syscall=yes
    ;;
  netbsd*)
    try_kqueue_syscall=yes
    ;;
  irix*)
    ;;
  linux*)
    enable_pthread_by_default=yes
    try_poll_syscall=yes
    try_epoll_syscall=yes
    try_inotify_syscall=yes
    ;;
  osf1*)
    ;;
  hpux*)
    ;;
  *mingw*)
    skip_iconv_check=yes
    ;;    
  cygwin*)
    ;;
  darwin*)
    PREFLAGS="$PREFLAGS -DOS_X -D_DARWIN_UNLIMITED_SELECT"
    enable_pthread_by_default=yes
    try_kqueue_syscall=yes

    # Although select() generally works as well as poll() on OS X,
    # getdtablesize() appears not to be constant within a process,
    # and that breaks static allocation of fd_sets
    try_poll_syscall=yes

    # -pthread is not needed and triggers a warning
    use_flag_pthread=no
    ;;
  nto-qnx*)
    use_flag_pthread=no
    ;;
  *)
    ;;
esac

############## C flags ################

AC_LANG_C

AC_TYPE_INTPTR_T
AC_TYPE_UINTPTR_T

AC_MSG_CHECKING([for getaddrinfo])
AC_TRY_LINK([#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>],
	[getaddrinfo(0, 0, 0, 0);],
   AC_DEFINE(HAVE_GETADDRINFO,1,[Have getaddrinfo])
   have_getaddrinfo=yes,
   have_getaddrinfo=no)
AC_MSG_RESULT($have_getaddrinfo)

iconv_lib_flag=""
if test "${skip_iconv_check}" = "no" ; then
 if test "${enable_iconv}" = "yes" ; then
  AC_CHECK_HEADER(iconv.h, enable_iconv=yes, enable_iconv=no)
  if test "${enable_iconv}" = "yes" ; then  
    # Does it all work, now?
    AC_TRY_RUN(
[   #include <iconv.h>]
[   #include <langinfo.h>]
    int main() {
[     iconv_open("UTF-8", "UTF-8");]
      return 0;
    }, enable_iconv=yes, enable_iconv=no, enable_iconv=yes)
    if test "${enable_iconv}" = "no" ; then  
      # Try adding -liconv ?
      #  We did not use AC_CHECK_LIB because iconv is sometimes macro-renamed
      ORIG_LIBS="$LIBS"
      LIBS="$LIBS -liconv"
      AC_TRY_RUN(
[     #include <iconv.h>]
[     #include <langinfo.h>]
      int main() {
[     iconv_open("UTF-8", "UTF-8");]
      return 0;
      }, enable_iconv=yes, enable_iconv=no, enable_iconv=yes)
      if test "${enable_iconv}" = "no" ; then
        LIBS="$ORIG_LIBS"
      else
        iconv_lib_flag=" -liconv"      
      fi
    fi
  fi
 fi
 [ msg="iconv is usable" ]
 AC_MSG_CHECKING($msg)
 iconv_usage_result="$enable_iconv$iconv_lib_flag"
 AC_MSG_RESULT($iconv_usage_result)
fi
if test "${enable_iconv}" = "no" ; then
  AC_DEFINE(RKTIO_NO_ICON,1,[Do not use iconv])
fi

if test "${enable_iconv}" = "yes" ; then
  AC_MSG_CHECKING([for nl_langinfo (CODESET)])
  AC_TRY_LINK([#include <langinfo.h>],
              [char *codeset = nl_langinfo (CODESET);],
              AC_DEFINE(RKTIO_HAVE_CODESET,1,[Have nl_langinfo (CODESET)])
               have_codeset=yes,
              have_codeset=no)
  AC_MSG_RESULT($have_codeset)
fi

[ msg="for mbsrtowcs" ]
AC_MSG_CHECKING($msg)
AC_LINK_IFELSE([AC_LANG_SOURCE([
   #include <wchar.h>
   #include <strings.h>
   int main() {
     mbstate_t state;
     char *src = "X";
     bzero(&state, sizeof(mbstate_t));
     mbsrtowcs(0, &src, 0, &state);
     return 0;
   }])], mbsrtowcs=yes, mbsrtowcs=no)
if test "$mbsrtowcs" = "no" ; then
  CFLAGS="$CFLAGS -DNO_MBTOWC_FUNCTIONS"
fi
AC_MSG_RESULT($mbsrtowcs)

if test "${try_poll_syscall}" = "yes" ; then
  [ msg="for poll" ]
  AC_MSG_CHECKING($msg)
  AC_LINK_IFELSE([AC_LANG_SOURCE([
     #include <poll.h>
     int main() {
      struct pollfd pfd;
      int r;
      pfd.fd = 0;
      pfd.events = POLLIN;
      r = poll(&pfd, 1, 0);
      return 0;
     }])], use_poll=yes, use_poll=no)
  AC_MSG_RESULT($use_poll)
  if test "${use_poll}" = "yes" ; then
     AC_DEFINE(HAVE_POLL_SYSCALL,1,[Have poll])
  fi
fi

if test "${try_epoll_syscall}" = "yes" ; then
  [ msg="for epoll" ]
  AC_MSG_CHECKING($msg)
  AC_LINK_IFELSE([AC_LANG_SOURCE([
    #include <sys/epoll.h>
    int main() {
      int fd;
      struct epoll_event ev;
      fd = epoll_create(5);
      ev.events = EPOLLIN | EPOLLONESHOT;
      epoll_ctl(fd, EPOLL_CTL_ADD, 0, &ev);
      return 0;
    }])], use_epoll=yes, use_epoll=no)
  AC_MSG_RESULT($use_epoll)
  if test "${use_epoll}" = "yes" ; then
     AC_DEFINE(HAVE_EPOLL_SYSCALL,1,[Have epoll])
  fi
fi

if test "${try_inotify_syscall}" = "yes" ; then
  [ msg="for inotify" ]
  AC_MSG_CHECKING($msg)
  AC_LINK_IFELSE([AC_LANG_SOURCE([
    #include <sys/inotify.h> 
    int main() {
      int fd;
      int wd;
      fd = inotify_init();
      wd = inotify_add_watch(fd, "/tmp", 
                             (IN_CREATE | IN_DELETE | IN_DELETE_SELF
                              | IN_MODIFY | IN_MOVE_SELF | IN_MOVED_TO));
      return 0;
    }])], use_inotify=yes, use_inotify=no)
  AC_MSG_RESULT($use_inotify)
  if test "${use_inotify}" = "yes" ; then
     AC_DEFINE(HAVE_INOTIFY_SYSCALL,1,[Have inotify])
  fi
fi

if test "${try_kqueue_syscall}" = "yes" ; then
  [ msg="for kqueue" ]
  AC_MSG_CHECKING($msg)
  AC_LINK_IFELSE([AC_LANG_SOURCE([
     #include <sys/types.h>
     #include <sys/event.h>
     #include <sys/time.h>
     int main() {
        int kq;
        struct kevent kev;
        struct timespec timeout = {0, 0};
        kq = kqueue();
        EV_SET(&kev, 0, EVFILT_READ, EV_ADD, 0, 0, NULL);
        kevent(kq, &kev, 1, NULL, 0, &timeout);
        return 0;
     }])], use_kqueue=yes, use_kqueue=no)
  AC_MSG_RESULT($use_kqueue)
  if test "${use_kqueue}" = "yes" ; then
     AC_DEFINE(HAVE_KQUEUE_SYSCALL,1,[Have kqueue])
  fi
fi

LFS_CFLAGS=`getconf LFS_CFLAGS 2> /dev/null`
if test "${LFS_CFLAGS}" != "" && test "${LFS_CFLAGS}" != "undefined"; then
  echo "Large-file support: ${LFS_CFLAGS}"
  CFLAGS="${CFLAGS} ${LFS_CFLAGS}"  
fi

############### pthread ###################

if test "${enable_pthread}" = "" ; then
  if test "${enable_pthread_by_default}" = "yes" ; then
    enable_pthread=yes
  fi
fi

if test "${enable_pthread}" = "yes" ; then
  if test "${use_flag_pthread}" = "yes" ; then
    PREFLAGS="$PREFLAGS -pthread"
    LDFLAGS="$LDFLAGS -pthread"
  fi
  if test "${use_flag_posix_pthread}" = "yes" ; then
    PREFLAGS="$PREFLAGS -D_POSIX_PTHREAD_SEMANTICS -D_REENTRANT"
  fi
  AC_DEFINE(RKTIO_USE_PTHREADS, 1, [Pthread enabled])
fi

############## libtool ################

if test "${enable_shared}" = "yes" ; then
  echo "Configuring libtool"

  if test ! -d "../lt" ; then
    mkdir "../lt"
  fi
  abssrcdir=`(cd ${srcdir}; pwd)`

  if test "${LIBTOOLPROG}" = "" ; then  
    (cd ../lt; sh ${abssrcdir}/../lt/configure --enable-shared --disable-static)
    LIBTOOLPROG=`pwd`/../lt/libtool
  fi

  if test "${need_gcc_static_libgcc}" = "yes" ; then
    need_gcc_static_libgcc=""
    if test "$CC" = "gcc" ; then 
      gcc_vers_three=`${CC} -v 2>&1 | grep "version 3[.]"`
      if test "$gcc_vers_three" = "" ; then
        need_gcc_static_libgcc=""
      else
        need_gcc_static_libgcc=" -XCClinker -static-libgcc"
      fi
    fi
  fi

  AR="${LIBTOOLPROG} --mode=link --tag=CC $CC${need_gcc_static_libgcc}${ar_libtool_no_undefined} -release ${plt_lib_version} \$(ARLIBFLAGS) -o"
  STATIC_AR="${LIBTOOLPROG} --mode=link --tag=CC $CC -o"
  ARFLAGS=""
  RANLIB=":"
  RKTLINKER="${LIBTOOLPROG} --mode=link --tag=CC $CC${need_gcc_static_libgcc}"
  CC="${LIBTOOLPROG} --mode=compile --tag=CC $CC"
  LIBSFX=la
  LTO="lo"
  LTA="la"
  STRIP_LIB_DEBUG=":"
else
  LIBSFX=a
  ICP=cp
  ICP_LIB=cp
  LTO="o"
  LTA="a"
  RKTLINKER='$(CC)'
  STATIC_AR="$AR"
fi

############## final output ################

AC_SUBST(CC)
AC_SUBST(CFLAGS)
AC_SUBST(LDFLAGS)
AC_SUBST(RKTLINKER)
AC_SUBST(AR)
AC_SUBST(ARFLAGS)
AC_SUBST(RANLIB)
AC_SUBST(STATIC_AR)

AC_SUBST(LIBSFX)
AC_SUBST(LTO)
AC_SUBST(LTA)

makefiles="Makefile"

AC_OUTPUT($makefiles)
