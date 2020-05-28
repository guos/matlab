# matlab



#!/bin/sh
#
#  Name:
#
#     lmboot	starts the FLEXnet License Manager at boot time.
#     
#  Usage:
#
#     lmboot 	-s | [-h|-help] [-arch] [-k] [-e [nlines]] [-v] 
#				[-wait secs] [-c licensefile]
#			        [-l debuglog] [-u username] [-2 -p]
#			        [-x lmdown|lmremove] [lmgrd_options ...]
#
#  Description:
#
#     This Bourne Shell script attempts to start up the FLEXnet license
#     manger.
#
#     In all cases:
#
#	  1. It will never start up as root user.
#         2. The debug log output will be piped to a simple shell
#	     script that will allow the logfile to be switched
#	     without affecting the proper operation of the license
#	     manager.
#     
#     It uses configuration information found normally in the file,
#
#                      $MATLAB/etc/lmopts.sh  
#                              or 
#                      $LM_ROOT/lmopts.sh     (standalone)
#
#     Command line arguments can override some of this configuration.
#
#     MathWorks script mode:
#
#         A new LM_START_FILE is created which allows for 'lmdown' to take
#         down the license manager reliably. It is designed to be flexible
#         and allow for inclusion of vendor daemons other than for just
#         MATLAB.
#	  
#		. ln -s $LM_ROOT/$ARCH/lmgrd	$LM_RUN_DIR/$LM_MARKER.ld
#	 	. cp -p $LM_FILE		$LM_RUN_DIR/$LM_MARKER.dat
#               . Given a license file of the form:
#----------------------------------------------------------------------------
#     #. . .
#     SERVER . . .
#     SERVER . . .
#     . . .
#     DAEMON name1 daemonfile optionsfile
#     DAEMON name2 daemonfile optionsfile
#     . . .
#     INCREMENT . . .
#     . . .
#----------------------------------------------------------------------------
#     		   create a license file of the form
#----------------------------------------------------------------------------
#     #. . .
#     SERVER . . .
#     SERVER . . .
#     . . .
#     DAEMON name1 $LM_RUN_DIR/$LM_MARKER.vd1 optionsfile
#     DAEMON name2 $LM_RUN_DIR/$LM_MARKER.vd2 optionsfile
#     . . .
#     INCREMENT . . .
#     . . .
#----------------------------------------------------------------------------
#     		  This requires links to me made.  The FORM of the links is
#		  critical. They must access the actual executable daemons,
#		  NOT scripts! This is so that the entries in the process
#		  table get the proper names. So for example, on the sol2 we
#		  might have
#
#     		      name1 = MLM
#
#	ln -s $LM_ROOT/sol2/MLM  $LM_RUN_DIR/$LM_MARKER.vd1
#
#     		      name2 = PVI 
#
#       ln -s $PVI/license/bin/bin.sun4/PVI $LM_RUN_DIR/$LM_MARKER.vd2
#
#
#
#  Options:
#
#     -s		- Create a symbolic link
#
#			      /etc/lmboot_TMW -> $LM_ROOT/lmboot 
#			      /etc/lmdown_TMW -> $LM_ROOT/lmdown
#
#     -h | -help 	- Help. Print usage.
#
#     -arch             - Assume local host has architecture arch.
#
#     -k		- Check that the tcp ports in the license
#		          file are free on the local host.
#
#     -e nlines		- List the last nlines of the debug logfile(s).
#			  The default is 10.
#
#     -v                - Verbose listing. Outputs the configuration
#			  information.
#
#     -wait secs	- Wait for secs seconds for MathWorks
#			  vendor daemon to come up. A minimum of 3
#			  seconds and a maximum of 300 seconds are
#			  enforced.
#		          DEFAULT: 3 seconds
#
#     -c licensefile    - Path to license file.
#			  DEFAULT: $MATLAB/etc/license.dat
#                                  $LM_ROOT/license.dat (standalone)
#
#     -l debuglog       - Path to FLEXnet debug logfile.
#			  DEFAULT: /var/tmp/lm_TMW.log
#
#     -u username       - Username to start/stop license manager.
#                         Only useful at machine bootup time or if you
#                         are running as superuser.
#                         DEFAULT: none
#
#     -2 -p 		- Restricts usage of lmdown, lmreread, and 
#			  lmremove to a FLEXnet administrator who is by 
#			  default root. If there a Unix group called 
#			  `lmadmin' then use is restricted to only 
#			  members of that group. If root is not a member
#			  of this group, then root does not have permission
#			  to use any of the above utilities. 
#
#			  NOTE: This option only restricts the binary versions
#			  of the utilities listed above that are normally 
#			  shipped with any FLEXnet distribution.  MathWorks
#			  ships these binary files (usually in 
#			  $MATLAB/etc/$ARCH) and shell script wrapper functions
#			  (usually in $MATLAB/etc/) that use the binary 
#			  utilities.  When run on the machine that is running 
#			  the license manager, the shell script wrapper 
#			  functions can still bring down the license manager if
#			  they are run by the same user that started the 
#			  license manager, or if they are run by the root user.
#			  The binary versions will not bring down the license 
#			  manager when this option was used with lmstart.
#
#     -x lmdown		- Disallow the lmdown command (no user can run lmdown). 
#			  The note for option -2 -p also applies to this option. 
#
#     -x lmremove	- Disallow the lmremove command (no user can run lmremove).
#			  The note for option -2 -p also applies to this option. 
#
#     lmgrd_options     - Additional 'lmgrd' options to augment
#			  those of LM_ARGS_LMGRD. Do not use -z.
#
# note(s):
#
#     1. This script executes without changing the current directory.
#        This means that relative pathnames can be used in command
#        options.
#
#  Copyright 1986-2018 The MathWorks, Inc.
#-----------------------------------------------------------------------
#23456789012345678901234567890123456789012345678901234567890123456789012
#
    arg0_=$0
#
    trap "rm -rf /tmp/$$a > /dev/null 2>&1; exit 1" 1 2 3 15
#
# Initialize if license manager environment has not been set yet
#
    if [ "$LM_SET_ENV" = "" -a "$ARCH" != "" ]; then
        ARCH=""
    fi
#
#========================= archlist.sh (start) ============================
#
# usage:        archlist.sh
#
# abstract:     This Bourne Shell script creates the variable ARCH_LIST.
#
# note(s):      1. This file is always imbedded in another script
#
# Copyright 1997-2013 The MathWorks, Inc.
#----------------------------------------------------------------------------
#
    ARCH_LIST='glnxa64 maci64'
#=======================================================================
# Functions:
#   check_archlist ()
#=======================================================================
    check_archlist () { # Sets ARCH. If first argument contains a valid
			# arch then ARCH is set to that value else
		        # an empty string. If there is a second argument
			# do not output any warning message. The most
			# common forms of the first argument are:
			#
			#     ARCH=arch
			#     MATLAB_ARCH=arch
			#     argument=-arch
			#
                        # Always returns a 0 status.
                        #
                        # usage: check_archlist arch=[-]value [noprint]
                        #
	if [ $# -gt 0 ]; then
	    arch_in=`expr "$1" : '.*=\(.*\)'`
	    if [ "$arch_in" != "" ]; then
	        ARCH=`echo "$ARCH_LIST EOF $arch_in" | awk '
#-----------------------------------------------------------------------
	{ for (i = 1; i <= NF; i = i + 1)
	      if ($i == "EOF")
		  narch = i - 1
	  for (i = 1; i <= narch; i = i + 1)
		if ($i == $NF || "-" $i == $NF) {
		    print $i
		    exit
		}
	}'`
#-----------------------------------------------------------------------
	       if [ "$ARCH" = "" -a $# -eq 1 ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo "    Warning: $1 does not specify a valid architecture - ignored . . ."
echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	       fi
	    else
		ARCH=""
	    fi
	else
	    ARCH=""
	fi
#
	return 0
    }
#=======================================================================
#========================= archlist.sh (end) ==============================
#
#=======================================================================
# Functions:
#   scriptpath ()
#   lsbtest ()
#   bldstartfile ()
#   show_debuglogs ()
#   createlinks ()
#   help_cannotstart ()
#   standalone_lm ()
#   tail_debuglog ()
#   tcp_port_status ()
#   check_log_for_verification
#   waitfor_lockfile ()
#   waitfor_return ()
#=======================================================================
    scriptpath () { # Returns path of this script as a directory,
                    # ROOTDIR, and command name, CMDNAME.
		    #
		    # Returns a 0 status unless an error occurred.
		    #
                    # usage: scriptpath
                    #
#
	filename=$arg0_
#
# Now it is either a file or a link to a file.
#
        cpath=`pwd`
#
# Follow up to 8 links before giving up. Same as BSD 4.3
#
        n=1
        maxlinks=8
        while [ $n -le $maxlinks ]
        do
#
# Get directory correctly!
#
	    newdir=`echo "$filename" | awk '
                { tail = $0
                  np = index (tail, "/")
                  while ( np != 0 ) {
                      tail = substr (tail, np + 1, length (tail) - np)
                      if (tail == "" ) break
                      np = index (tail, "/")
                  }
                  head = substr ($0, 1, length ($0) - length (tail))
                  if ( tail == "." || tail == "..")
                      print $0
                  else
                      print head
                }'`
	    if [ ! "$newdir" ]; then
	        newdir="."
	    fi
	    if [ ! -d $newdir ]; then
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ''
echo 'Internal error 1: Could not determine the path of the command.'
echo ''
echo "                  original command path = $arg0_"
echo "                  current  command path = $filename"
echo ''
echo '                  Please contact:'
echo '' 
echo '                      MathWorks Technical Support'
echo ''
echo '                  for further assistance.'
echo ''
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	        return 1
	    fi
	    cd $newdir
#
# Need the function pwd - not the built in one
#
	    newdir=`/bin/pwd`
#
	    newbase=`expr //$filename : '.*/\(.*\)' \| $filename`
            lscmd=`ls -l $newbase 2>/dev/null`
	    if [ ! "$lscmd" ]; then
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ''
echo 'Internal error 2: Could not determine the path of the command.'
echo ''
echo "                  original command path = $filename"
echo "                  current  command path = $filename"
echo ''
echo '                  Please contact:'
echo '' 
echo '                      MathWorks Technical Support'
echo ''
echo '                  for further assistance.'
echo ''
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	        return 1
	    fi
#
# Check for link portably
#
	    if [ `expr "$lscmd" : '.*->.*'` -ne 0 ]; then
	        filename=`echo "$lscmd" | awk '{ print $NF }'`
	    else
#
# It's a file
#
	        dir="$newdir"
	        CMDNAME="$newbase"
	        LM_ROOTdefault=`/bin/pwd`
	        break
	    fi
	    n=`expr $n + 1`
        done
        if [ $n -gt $maxlinks ]; then
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ''
echo 'Internal error 3: More than $maxlinks links in path to'
echo "                  this script. That's too many!"
echo ''
echo "                  original command path = $filename"
echo "                  current  command path = $filename"
echo ''
echo '                  Please contact:'
echo '' 
echo '                      MathWorks Technical Support'
echo ''
echo '                  for further assistance.'
echo ''
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    return 1
        fi
        cd $cpath
        return 0
    }
#=======================================================================
    lsbtest () { # Check if system is LSB compliant by running lmgrd.
                 # Input: Full path to lmgrd
                 # Return 0 for success, 1 for failure.
        lmgrdFile=$1

        # First check that file actually exists
        if [ -e $lmgrdFile ]; then
            # Try to run the file (with -v) to see if that fails or not.
            # A failure could be any problem, but it is likely that the 
            # system does not have the correct LSB packages installed.
            # This normally manifests itself as a "Command not found" error.
            "$lmgrdFile" -v > /dev/null 2>&1
            if [ $? -ne 0 ]; then
                echo ' '
                echo "    Error: Cannot run the file $lmgrdFile.  This may not be an LSB compliant system."
                return 1;
            fi
        else
            echo ' '
            echo "    Error: Cannot find file $lmgrdFile"
            return 1;
        fi

        return 0;
    }
#=======================================================================
    bldstartfile () { # This reads the license file and attempts to
		      # create a new license file with the daemon paths
		      # replaced by links. Each daemon path must be an
		      # executable AND not a shell script. Otherwise,
		      # it is ignored.
		      #
		      # Creates file:  $LM_START_FILE
	              # Creates links: $LM_RUN_DIR/$LM_MARKER.vd*
		      #                $LM_RUN_DIR/$LM_MARKER.ld
		      #
		      # Criterion for executable: output of the file
		      #     command has the word 'executable' in it AND
		      #     does not contain word 'shell'.
		      # Note: This is good enough. It has been tried on
		      #       all hosts
		      #
                      # Returns a 0 status unless an error occurred.
                      #
                      # usage: bldstartfile
                      #
#
# Create temporary license.dat file by copying and modifying the original file
#
        rm_if_owner $LM_START_FILE
        if [ $? -ne 0 ]; then
	    return 1
	fi
	bld_empty_file $LM_START_FILE
#
        cat $LM_FILE | fixmacendings | combine_cont_lines | \
	(ndaemon=0
         bad=0
         while read line
         do
	     field1=`echo $line | awk '{print $1}'`
             if [ "$field1" = "DAEMON" ]; then
	         daemonName=`echo $line | awk '{print $2}'` # field 2
	         daemonPath=`echo $line | awk '{print $3}'` # field 3
		 if [ "$daemonPath" = "" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo "    Error: DAEMON line missing daemon name or daemon path . . ."
echo ' '
echo "           $line"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
		     bad=1
		     continue
		 fi
	         case "$daemonName" in
# MATLAB
		     MLM)
	    	         ndaemon=`expr $ndaemon + 1`
        	 	 rm_if_owner $LM_RUN_DIR/$LM_MARKER.vd$ndaemon
        	         if [ $? -ne 0 ]; then
		            return 1
		         fi
	bld_sym_link $LM_ROOT/$ARCH/MLM $LM_RUN_DIR/$LM_MARKER.vd$ndaemon
		        ;;
# Other daemons
		     *)
        	         filePath=`actualpath $daemonPath`
        	         if [ "$filePath" = "" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo "    Error: Path for daemon ($daemonName) does not exist. Skipped . . ."
echo ' '
echo "           daemonPath = $daemonPath"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
			     bad=1
			     continue
		         else
			     s=`file $filePath | awk '
#-----------------------------------------------------------------------
		{ for (i = 1; i <= NF; i = i + 1)
		    if ($i ~ "executable,?$")
			e = 1
		    else if ($i ~ "shell,?$")
			s = 1
		  if (e && ! s) print e
		}'`
#-----------------------------------------------------------------------
			     if [ "$s" = "" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo "    Error: Path to daemon ($daemonName) must be executable AND"
echo "           not a shell script. Replace by new path to executable"
echo "           in license file."
echo ' '
echo "		 Skipped . . ."
echo ' '
echo "           daemonPath = $daemonPath"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
			         bad=1
			         continue
			     else
	    	    	         ndaemon=`expr $ndaemon + 1`
        	 	 	 rm_if_owner $LM_RUN_DIR/$LM_MARKER.vd$ndaemon
        	                 if [ $? -ne 0 ]; then
		            	    return 1
		                 fi
	bld_sym_link $daemonPath $LM_RUN_DIR/$LM_MARKER.vd$ndaemon
			     fi
		         fi
		         ;;
	         esac    
	         echo "$line" | \
 sed "s%$daemonPath%$LM_RUN_DIR/$LM_MARKER.vd$ndaemon%" >> $LM_START_FILE
	     elif [ "$field1" = "SERVER" ]; then
	        echo "$line" >> $LM_START_FILE
	     else
	         echo "$line" >> $LM_START_FILE
	     fi
	 done
         if [ "$bad" = "1" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
#
# Do not delete the LM_START_FILE so that you can look at it in case of
# an error.
#
             rm -f $LM_RUN_DIR/$LM_MARKER.vd*
         fi
	)
	if [ -f $LM_RUN_DIR/$LM_MARKER.vd1 -o -L $LM_RUN_DIR/$LM_MARKER.vd1 ]; then
#
# Create link for lmgrd.
#
	    rm_if_owner $LM_RUN_DIR/$LM_MARKER.ld
            if [ $? -ne 0 ]; then
	        return 1
	    fi
	    bld_sym_link $LM_ROOT/$ARCH/lmgrd $LM_RUN_DIR/$LM_MARKER.ld
            return 0
	else
	    return 1
	fi
    }
#=======================================================================
    show_debuglogs () { # List the last nlines of debug logfiles
			# LM_LOGFILE and LM_LOGFILE_REDUNDANT. Just
			# Print one if they are the same and mention
			# that the other is identical.
			#
			# Return 0 if you can print them else 1.
			#
			# usage: show_debuglogs nlines
			#
	nlines=$1
	bad=0
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo "LM_LOGFILE - $LM_LOGFILE . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
        check_readfile $LM_LOGFILE
	if [ $? -eq 0 ]; then
	    tail_debuglog 1 $nlines
	else
	    bad=1
	fi
#
	if [ "$LM_LOGFILE" != "$LM_LOGFILE_REDUNDANT" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo "LM_LOGFILE_REDUNDANT - $LM_LOGFILE_REDUNDANT . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
            check_readfile $LM_LOGFILE_REDUNDANT
	    if [ $? -eq 0 ]; then
	        tail_debuglog 3 $nlines
	    else
	        bad=1
	    fi
	else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo 'LM_LOGFILE_REDUNDANT is the same as LM_LOGFILE . . .'
echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	fi
	if [ "$bad" != "0" ]; then
	    return 1
	else
	    return 0
	fi
    }
#=======================================================================
    createlinks () { # Attempts to create links
		     #     /etc/lmboot_TMW -> $LM_ROOT/lmboot
		     #     /etc/lmdown_TMW -> $LM_ROOT/lmdown
		     # 
                     # Returns a 0 status unless an error occurred.
                     #
                     # usage: createlinks
                     #
	bad=0
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	if [ -f /etc/lmboot_TMW ]; then
	    rm -f /etc/lmboot_TMW > /dev/null 2>&1
	    if [ -f /etc/lmboot_TMW ]; then 
	        bad=`expr $bad + 1`
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    Error: cannot create /etc/lmboot_TMW link . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    else
		ln -s $LM_ROOT/lmboot /etc/lmboot_TMW
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    /etc/lmboot_TMW -> $LM_ROOT/lmboot link created . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    fi
	else
	    ln -s $LM_ROOT/lmboot /etc/lmboot_TMW > /dev/null 2>&1
	    if [ $? -ne 0 ]; then
	        bad=`expr $bad + 1`
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    Error: cannot create /etc/lmboot_TMW link . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    /etc/lmboot_TMW -> $LM_ROOT/lmboot link created . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    fi
        fi
#
	if [ -f /etc/lmdown_TMW ]; then
	    rm -f /etc/lmdown_TMW > /dev/null 2>&1
	    if [ -f /etc/lmdown_TMW ]; then 
	        bad=`expr $bad + 1`
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    Error: cannot create /etc/lmdown_TMW link . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    else
		ln -s $LM_ROOT/lmdown /etc/lmdown_TMW
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    /etc/lmdown_TMW -> $LM_ROOT/lmdown link created . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    fi
	else
	    ln -s $LM_ROOT/lmdown /etc/lmdown_TMW > /dev/null 2>&1
	    if [ $? -ne 0 ]; then
	        bad=`expr $bad + 1`
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    Error: cannot create /etc/lmdown_TMW link . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo '    /etc/lmdown_TMW -> $LM_ROOT/lmdown link created . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    fi
        fi
	if [ $bad -ne 0 ]; then
	    if [ $bad -eq 1 ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo ' '
    echo "           LM_ROOT = $LM_ROOT"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    fi
	    issuperuser
	    if [ $? -ne 0 ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo ' '
    echo '    NOTE: You are required to be superuser for this operation.'
    echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    fi
	    return 1
        else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo ' '
    echo "           LM_ROOT = $LM_ROOT"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    return 0    
	fi
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    }
#=======================================================================
    help_cannotstart () { # Output the tail of the debug logfile and then
			  # a help message with suggestions about what to
			  # do when the license manager doesn't start.
		          # 
                          # Always returns a 0 status.
                          #
                          # usage: help_cannotstart
                          #
	tail_debuglog $LM_NSERVERS 10
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ''
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	echon 'Please hit return for more help . . . > '
	read ans
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "------------------------------------------------------------------------"
echo "Notes: 1. To print more of the debug logfile use the '-e nlines' option"
echo "          on lmstart or lmboot commands. Use the -help option on any"
echo "          command for help."
echo "       2. To check that the daemons are up run the 'lmstat -a' command."
echo "       3. Use the '-wait secs' option on lmstart or lmboot commands"
echo "          to increase the wait time in batch."
echo "       4. If you are getting 'socket bind' errors in the logfile output"
echo "          first run 'lmdown' and then 'lmstart -k' to verify that all"
echo "          ports are clear. This is CRITICAL when using redundant"
echo "          servers. You must get nothing back from 'lmstart -k' on all"
echo "          servers before you are ready to run 'lmstart' to restart the"
echo "          license manager. This can take up to 5 minutes. If some of"
echo "          the ports do not clear in this time then they could be in "
echo "          use and you should change the port number on the SERVER line"
echo "          in your license file."
echo "------------------------------------------------------------------------"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	return 0
    }
#=======================================================================
    standalone_lm () { # If install_matlab does not exist in the
		       # parent directory then it is standalone
		       # and output a 1 else 0.
		       #
		       # Always returns 0 status.
                       #
                       # usage: standalone_lm
                       #
	if [ ! -f $LM_ROOTdefault/../install_matlab ]; then
	    echo 1
	else
	    echo 0
	fi
	return 0
    }
#=======================================================================
    tail_debuglog () { # Outputs at most the last num lines of the debug
		       # logfile with line number. The file always exists.
		       # 
                       # Always returns a 0 status.
                       #
                       # usage: tail_debuglog nservers num
                       #
	nservers=$1
	num=$2
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "------------------------------------------------------------------------"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	if [ "$nservers" != "3" ]; then 
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "`date`    LM_LOGFILE (last $num lines)"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    grep -n '.*' $LM_LOGFILE | tail -n $num | sed -e 's/:/ /'
	else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "`date`    LM_LOGFILE_REDUNDANT (last $num lines)"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    grep -n '.*' $LM_LOGFILE_REDUNDANT | tail -n $num | sed -e 's/:/ /'
	fi
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "------------------------------------------------------------------------"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	return 0
    }
#=======================================================================
    tcp_port_status () { # Does a netstat -a and checks for port usage.
		         # A port is in use if the entry has the
			 # the port number in it taken from the SERVER
			 # line of the license file. It prints out 
			 # anything in use.
			 #
			 # Example of netstat -s output:
			 #
			 # tcp        0      0 *:1705                 *:*                    LISTEN        martin    
			 #
			 # Returns a 0 status if no entries else 1.
			 #
                         # usage: tcp_port_status
                         #
	(netstat -a) > /tmp/$$a 2>&1
	if [ $? -ne 0 ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    echo ' '
    echo "    Error: 'netstat' command not on your search path . . ."
    echo '           Please fix and try again.'
    echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    rm -rf /tmp/$$a
	    return 1
	fi
#
# Produce '(\.num1|:num1|...)' for portlist
#
	portlist=`getserver_ports | awk '
#-----------------------------------------------------------------------
    BEGIN { backslash = sprintf ("%c", 92)   # set backslash
	  }  
	{ s = "("
	  for (i = 1; i <= NF; i = i + 1)
		s = s backslash "." $i "|" ":" $i "|"
	  l = length(s)
	  printf "%s", substr(s,1,l-1) ")"
	}'`
#-----------------------------------------------------------------------
        if [ "$portlist" = "" ]; then
	    return 1
	else
	    egrep $portlist /tmp/$$a > /dev/null 2>&1
            if [ $? -eq 0 ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "------------------------------------------------------------------------"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    egrep $portlist /tmp/$$a
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "------------------------------------------------------------------------"
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
		rm -f /tmp/$$a
		return 1
	    else
		rm -f /tmp/$$a
		return 0
	    fi
	fi
    }
#=======================================================================
    check_log_for_verification () { # Check if the liecnse verification ran.
                                    # Returns 0 for success (found that 
                                    # license verification ran).
                                    # Returns 1 for failure (license verification
                                    # has not run yet).
           # Look for the license verification string in the log file to 
           # see if it ran yet.
           verificationRan=`awk 'BEGIN { foundVerifier = 0 }  
                        /License verification/ { foundVerifier = 1}
                        /Server started on/ { foundVerifier = 0}
                        END {print foundVerifier}' $LM_LOGFILE`
                
           if [ $verificationRan -eq 1 ]; then
                # Return success
                return 0 
           fi
           # Return failure
           return 1
    }
#=======================================================================
    waitfor_lockfile () { # Waits for the vendor daemon lock file to
                          # appear
                          # 
                          # Returns a 0 status unless an error occurred.
                          #
                          # usage: waitfor_lockfile
                          #
        if [ $LM_NSERVERS -eq 3 ]; then
            # Let redundant servers sleep a little longer to avoid
            # a race condition with lmgrd/MLM which are running in 
            # the background.  Redundant servers take more time to 
            # start up since they need to connect to other servers,
            # especially when restarting a stopped node.
            sleep 6
        else
            sleep 3
        fi
        if [ ! -f $LM_LOCKFILE ]; then
            waitfor_return
            if [ $? -ne 0 ]; then 
                return 1
            fi
        fi

        return 0
    }
#=======================================================================
    waitfor_return () { # Waits for the MATLAB vendor daemon lock file
			# to appear.
		        # 
                        # Returns a 0 status unless an error occurred.
                        #
                        # usage: waitfor_return
                        #

        # First do an early check for the license verification text in the log.
        # Do this here first so that we don't bother printing out any
        # messages about waiting for the vendor daemon.  If we find the text,
        # then there was a failure (since there is no lock file).  If we don't
        # find what we are looking for, then we continue on with the normal 
        # checks (which will check the log again).
        if [ ! -f $LM_LOCKFILE ]; then
            check_log_for_verification
            if [ $? -eq 0 ]; then
                # return failure
                return 1
            fi
        fi

	MIN_DAEMON_WAIT=3
	MAX_DAEMON_WAIT=300
#
# Redundant servers, wait for vendor daemon lock file to appear
#
	wait_time=3
	mesg_time=3
#
# Give up after max(MIN_DAEMON_WAIT, min(LM_DAEMON_WAIT, MAX_DAEMON_WAIT))
# seconds
#
	wait_limit=`expr $LM_DAEMON_WAIT + 0`
	if [ $wait_limit -lt $MIN_DAEMON_WAIT ]; then
	    wait_limit=$MIN_DAEMON_WAIT
	elif [ $wait_limit -gt $MAX_DAEMON_WAIT ]; then
	    wait_limit=$MAX_DAEMON_WAIT
	fi
#
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "    Waiting $wait_limit secs for MATLAB vendor daemon to come up . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	interactive
	if [ $? -eq 0 ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo '            Type your interrupt character (usually CTRL-C) to quit.'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	fi
#
	if [ $wait_time -ge $wait_limit ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "    Time = $wait_time secs  :  time limit reached . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	    interactive
	    if [ $? -eq 0 ]; then
		help_cannotstart
	    fi
	    return 1
	else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "    Time = $wait_time secs  :  still waiting . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	fi
	trap "help_cannotstart; return 1" 1 2 3 15
#
	while true
	do
	    if [ ! -f $LM_LOCKFILE ]; then
                # Check for the license verification text in the log.
                # If we found it, and there is no lock file, then there must
                # have been an error.  If we didn't find it, then continue.
                check_log_for_verification
                if [ $? -eq 0 ]; then
                    # return failure
                    return 1
                fi
		wait_time=`expr $wait_time + 3`
		mesg_time=`expr $mesg_time + 3`
#
		if [ $wait_time -ge $wait_limit ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "    Time = $wait_time secs  :  time limit reached . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
		    interactive
		    if [ $? -eq 0 ]; then
		        help_cannotstart
		    fi
		    return 1
		fi
		if [ "$mesg_time" = "30" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo "    Time = $wait_time secs  :  still waiting . . ."
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
		    interactive
		    tail_debuglog $LM_NSERVERS 2
		    mesg_time=0
		fi
		sleep 3
	    else
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo '    MATLAB vendor daemon is up . . .'
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
		break
	    fi
	done
	return 0
    }
#=======================================================================
#
# Determine the default license manager root directory (LM_ROOTdefault)
# and the current directory (cpath)
#
    scriptpath
    if [ $? -ne 0 ]; then
        exit 1
    fi
#
# Verify input
#
    stat="OK"
    msg=""
#
    linkonly=
#
    help=
    check_tcp_ports=
    list_debuglog=
    list_nlines=10
    verbose=
    LM_SWITCHMODE=
    licensefile=
    logfile=
    username=
    daemon_wait=
    arglist_lmgrd=
#
    nargs=$#
#
    while [ "$stat" = "OK" -a $# -gt 0 ]; do
	case "$1" in
	    -h|-help)		# -help: Help option.
	        help=1
		;;
	    -s)
		linkonly=1
		if [ $nargs -eq 1 ]; then
		    :
		elif [ $# -gt 1 ]; then
		    if [ `expr "$2" : '-.*'` -gt 0 ]; then
msg='-s is either the only option or needs an argument for lmgrd'
			stat=""
		    else
                	arglist_lmgrd="$arglist_lmgrd $1"
		    fi
		else
		    stat=""
		fi
		;;
	    -k)
		check_tcp_ports=1
		;;
	    -e)
                list_debuglog=1
                if [ $# -gt 1 ]; then
#
# Check for number
#
                    if [ `expr "//$2" : '//-.*'` -eq 0 ]; then
                        t=`expr "//$2" : '//\([0-9]*\)$'`
                        if [ "$t" != "" ]; then
                            shift
                            list_nlines=$1
                        fi
                    fi
                fi
		;;
	    -v)
	        verbose=1
		;;
	    -w)
		LM_SWITCHMODE=1
		;;
	    -wait)
                if [ $# -eq 1 ]; then
		    msg='usage'
                    stat=""
                else
                    shift
#
# Check for number
#
		    t=`expr "//$1" : '//\([0-9]*\)$'`
		    if [ "$t" != "" ]; then
                        daemon_wait=$1
		    else
			msg='wait argument not a number'
			stat=""
		    fi
                fi
                ;;
	    -c)
                if [ $# -eq 1 ]; then
		    msg='usage'
                    stat=""
                else
                    shift
                    licensefile=$1
                fi
                ;;
	    -l)
                if [ $# -eq 1 ]; then
		    msg='usage'
                    stat=""
                else
                    shift
                    logfile=$1
                fi
		;;
	    -u)
                if [ $# -eq 1 ]; then
		    msg='usage'
                    stat=""
                else
                    shift
                    username=$1
                fi
		;;
	    -2) #-2 -p disables lmdown
                if [ $# -eq 1 ]; then
                    msg='usage'
                    stat=""
                else
                    if [ "$2" = "-p" ]; then
                        arglist_lmgrd="$arglist_lmgrd -2 -p"
                        shift
                    else
                        msg='usage'
                        stat=""
                    fi
                fi
                ;;
            -p)  #-p -2 option to disable lmdown (same as -2 -p)
                if [ $# -eq 1 ]; then
                    msg='usage'
                    stat=""
                else
                    if [ "$2" = "-2" ]; then
                        arglist_lmgrd="$arglist_lmgrd -2 -p"
                        shift
                    else
                        msg='usage'
                        stat=""
                    fi
                fi
                ;;
            -x)  #-x option to disable lmdown or lmremove
                if [ $# -eq 1 ]; then
                    msg='usage'
                    stat=""
                else
                    if [ "$2" = "lmdown" ]; then
                        arglist_lmgrd="$arglist_lmgrd -x lmdown"
                        shift
                    elif [ "$2" = "lmremove" ]; then
                        arglist_lmgrd="$arglist_lmgrd -x lmremove"
                        shift
                    else
                        msg='usage'
                        stat=""
                    fi
                fi
                ;;
	    -*)
                check_archlist argument=$1 noprint
                if [ "$ARCH" = "" ]; then
		    msg="-arch ($1) is invalid . . ."
		    stat=""
                fi
                ;;
	    *)
                arglist_lmgrd="$arglist_lmgrd $1"
                ;;
	esac
	shift
    done
#
    export help
    export verbose
    export LM_SWITCHMODE
    export daemon_wait
    export licensefile
    export logfile
    export username
    export arglist_lmgrd
#
    if [ "$linkonly" != "" -o "$list_debuglog" != "" ]; then
        ck_licensefile=0; export ck_licensefile
        ck_writelogfile=0; export ck_writelogfile
    else
        ck_licensefile=1; export ck_licensefile
        ck_writelogfile=1; export ck_writelogfile
    fi
#
    if [ "$stat" = "" -o "$help" != "" ]; then
	if [ "$msg" != "" ]; then
#-----------------------------------------------------------------------
    echo " "
    echo "    Error: $msg"
    echo " "
#-----------------------------------------------------------------------
	fi
#-----------------------------------------------------------------------
(
echo "------------------------------------------------------------------------"
echo " "
echo "    Usage: lmboot -s | [-h|-help] [-arch] [-k] [-e [nlines]] [-v]"
echo "                       [-wait secs] [-c licensefile] [-l debuglog]"
echo "                       [-u username] [-2 -p] [-x lmdown|lmremove]"
echo "                       [lmgrd_options ...]"
echo " "
echo "    -s                - Create symbolic links"
echo '                            /etc/lmboot_TMW -> $LM_ROOT/lmboot' 
echo '                            /etc/lmdown_TMW -> $LM_ROOT/lmdown'
#-----------------------------------------------------------------------
        if [ "`standalone_lm`" = "1" ]; then
#-----------------------------------------------------------------------
echo '                        where $LM_ROOT is the license manager root'
echo '                        directory.'
#-----------------------------------------------------------------------
	else
#-----------------------------------------------------------------------
echo '                        where $LM_ROOT is $MATLAB/etc.'
#-----------------------------------------------------------------------
	fi
#-----------------------------------------------------------------------
echo '                        You must be superuser.'
echo "    -h|-help          - Help."
echo "    -arch             - Assume local host has architecture arch."
echo "    -k                - Check that the tcp ports in the license"
echo "                        file are free on the local host."
echo "    -e nlines         - List the last nlines of the debug logfile(s)."
echo "                        DEFAULT: 10 lines"
echo "    -v                - Verbose listing. Outputs the configuration"
echo "                        information."
echo "    -wait secs        - Wait for secs seconds for MathWorks vendor"
echo "                        daemon to come up. A minimum of 3 seconds"
echo "                        and a maximum of 300 seconds are enforced."
echo "                        DEFAULT: 3 seconds"
echo "    -c licensefile    - Path to license file."
#-----------------------------------------------------------------------
        if [ "`standalone_lm`" = "1" ]; then
#-----------------------------------------------------------------------
echo '                        DEFAULT: $LM_ROOT/license.dat'
#-----------------------------------------------------------------------
	else
#-----------------------------------------------------------------------
echo '                        DEFAULT: $MATLAB/etc/license.dat'
#-----------------------------------------------------------------------
	fi
#-----------------------------------------------------------------------
echo "    -l debuglog       - Path to FLEXnet debug logfile."
echo '                        DEFAULT: /var/tmp/lm_TMW.log'
echo "    -u username       - Username to start/stop license manager."
echo "                        Only useful at machine bootup time or if you"
echo "                        are running as superuser."
echo "                        DEFAULT: none"
echo "    -2 -p             - Restricts usage of lmdown, lmreread, and"
echo "                        lmremove to a FLEXnet administrator who is by"
echo "                        default root. If there a Unix group called" 
echo "                        'lmadmin' then use is restricted to only" 
echo "                        members of that group. If root is not a member"
echo "                        of this group, then root does not have permission"
echo "                        to use any of the above utilities."
echo "                        NOTE: This option only restricts the binary versions"
echo "                        of the utilities listed above that are normally "
echo "                        shipped with any FLEXnet distribution.  MathWorks"
echo "                        ships these binary files (usually in "
echo '                        $MATLAB/etc/$ARCH) and shell script wrapper functions'
echo '                        (usually in $MATLAB/etc/) that use the binary '
echo "                        utilities.  When run on the machine that is running "
echo "                        the license manager, the shell script wrapper "
echo "                        functions can still bring down the license manager if"
echo "                        they are run by the same user that started the "
echo "                        license manager, or if they are run by the root user."
echo "                        The binary versions will not bring down the license "
echo "                        manager when this option was used with lmstart."
echo "    -x lmdown         - Disallow the lmdown command (no user can run lmdown)."
echo "                        The note for option -2 -p also applies to this option."
echo "    -x lmremove       - Disallow the lmremove command (no user can run lmremove)."
echo "                        The note for option -2 -p also applies to this option."
echo "    lmgrd_options     - Additional 'lmgrd' options to augment"
echo "                        those of LM_ARGS_LMGRD. Do not use -z."
echo " "
echo "    Attempts to start up the FLEXnet license manager. In all cases:"
echo "       1. It will never start up as superuser."
echo "       2. The debug log output will be piped to a simple shell"
echo "          script that will allow the logfile to be switched"
echo "          without affecting the proper operation of the license"
echo "          manager."
echo "    It uses configuration information found normally in the file,"
echo " "
#-----------------------------------------------------------------------
        if [ "`standalone_lm`" = "1" ]; then
#-----------------------------------------------------------------------
echo '                   $LM_ROOT/lmopts.sh.'
#-----------------------------------------------------------------------
	else
#-----------------------------------------------------------------------
echo '                   $MATLAB/etc/lmopts.sh.'
#-----------------------------------------------------------------------
	fi
#-----------------------------------------------------------------------
echo " "
echo "    Command line arguments can override some of this"
echo "    configuration information."
echo " "
echo "------------------------------------------------------------------------"
) > /tmp/$$a
#-----------------------------------------------------------------------
	if [ "$help" = "1" ]; then
	    verbose=1
	    . $LM_ROOTdefault/util/setlmenv.sh >> /tmp/$$a
	    more /tmp/$$a
	else
	    more /tmp/$$a
	fi
	rm -f /tmp/$$a
        exit 1
    fi
#
# Set the environment
#
    . $LM_ROOTdefault/util/setlmenv.sh
#
    if [ "$linkonly" != "" ]; then
#
# Create the links only
#
	createlinks
	exit $?
    fi
#
# Cannot start up license manager using port@host
#
    if [ "$LM_PORTATHOST" = "1" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo '    Error: Your license file is of the form port@host.'
echo '           You cannot start the license manager using this form of'
echo '           license file.' 
echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
	exit 1
    fi
#
# Check the local tcp port usage
#
    if [ "$check_tcp_ports" != "" ]; then
	tcp_port_status
        exit $?
    fi
#
# List the end of the debug logfile
#
    if [ "$list_debuglog" != "" ]; then
        show_debuglogs $list_nlines | more
        exit $?
    fi
#
    if [ "$LM_UTIL_MODE" != "1" ]; then
#
# Flexera mode
#
	issuperuser
	if [ $? -eq 0 ]; then
	    if  [ "$LM_ADMIN_USER" = "" -o "$LM_ADMIN_USER" = "root" -o \
		  "$LM_ADMIN_USER" = "0" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo '    Error: You are superuser.'
echo ' '
echo '           For security reasons superuser is not allowed to start'
echo '           the license manager.'
echo ' '
echo '           Either startup the license manager as a regular user or'
echo '           stay as superuser and use the -u option and an alternative'
echo '           username. For help use the -h option.'
echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
                exit 1
	    else
#
# start up the license manager using LM_ADMIN_USER
#
# Examples:
#     su username -c "umask 022; lmgrd -c license.dat -l log"
#     lmgrd -z | sh -c 'while read line; do echo "$line" >> LOG; done'
#
		LM_START_FILE=`actualpath $LM_START_FILE`
		LM_LOGFILE=`actualpath $LM_LOGFILE`
                incmd="$LM_ROOT/$ARCH/lmgrd -z $LM_ARGS_LMGRD -c $LM_START_FILE" 
    		outcmd="'"'while read line; do echo "$line" >> '$LM_LOGFILE"; done'"
    		su $LM_ADMIN_USER -c "umask 022; $incmd | sh -c $outcmd" &
#
# Check for the lockfile. If none, things really went bad.
#
		waitfor_lockfile
		if [ $? -ne 0 ]; then 
	    	    exit 1
		fi
		exit 0
	    fi
	fi
#
	LM_START_FILE=`actualpath $LM_START_FILE`
	LM_LOGFILE=`actualpath $LM_LOGFILE`
        incmd="$LM_ROOT/$ARCH/lmgrd -z $LM_ARGS_LMGRD -c $LM_START_FILE" 
    	outcmd="'"'while read line; do echo "$line" >> '$LM_LOGFILE"; done'"
    	eval "$incmd | sh -c $outcmd" &
#
# Check for the lockfile. If none, things really went bad.
#
	waitfor_lockfile
	if [ $? -ne 0 ]; then 
	    exit 1
	fi
	exit 0
    else
#
# MathWorks mode
#
	issuperuser
	if [ $? -eq 0 ]; then
	    if  [ "$LM_ADMIN_USER" = "" -o "$LM_ADMIN_USER" = "root" -o \
		  "$LM_ADMIN_USER" = "0" ]; then
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
echo ' '
echo '    Error: You are superuser.'
echo ' '
echo '           For security reasons superuser is not allowed to start'
echo '           the license manager.'
echo ' '
echo '           Either startup the license manager as a regular user or'
echo '           stay as superuser and use the -u option and an alternative'
echo '           username. For help use the -h option.'
echo ' '
#++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
                exit 1
	    else
		bldstartfile
		if [ $? -ne 0 ]; then
	    	    exit 1
		fi
#	
                # Use path to actual file, not to link, otherwise root can't access it
                lsbtest "$LM_ROOT/$ARCH/lmgrd"
                if [ $? -ne 0 ]; then
                    exit 1
                fi
#
		LM_START_FILE=`actualpath $LM_START_FILE`
		LM_LOGFILE=`actualpath $LM_LOGFILE`
#
# start up the license manager using LM_ADMIN_USER
#
                incmd="$LM_RUN_DIR/$LM_MARKER.ld -z $LM_ARGS_LMGRD -c $LM_START_FILE" 
    		outcmd="'"'while read line; do echo "$line" >> '$LM_LOGFILE"; done'"
    		su $LM_ADMIN_USER -c "umask 022; $incmd | sh -c $outcmd" &
#
# Check for the lockfile. If none, things really went bad.
#
		waitfor_lockfile
		if [ $? -ne 0 ]; then 
		    exit 1
		fi
		exit 0
	    fi
	fi
#
	bldstartfile
	if [ $? -ne 0 ]; then
	    exit 1
	fi
#
	lsbtest "$LM_RUN_DIR/$LM_MARKER.ld"
	if [ $? -ne 0 ]; then
	    exit 1
	fi
#
	LM_START_FILE=`actualpath $LM_START_FILE`
	LM_LOGFILE=`actualpath $LM_LOGFILE`
        incmd="$LM_RUN_DIR/$LM_MARKER.ld -z $LM_ARGS_LMGRD -c $LM_START_FILE" 
    	outcmd="'"'while read line; do echo "$line" >> '$LM_LOGFILE"; done'"
    	eval "$incmd | sh -c $outcmd" &
#
# Check for the lockfile. If none, things really went bad.
#
	waitfor_lockfile
	if [ $? -ne 0 ]; then 
	    exit 1
	fi
	exit 0
    fi

