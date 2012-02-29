Under Siege
===========

Under Siege (us_*) is a collection of perl scripts for setting up and running 
load-tests using the [Siege load-testing program](http://www.joedog.org/siege-home/).
Siege has the ability to take as input a list of URLs to test. We can use this 
(with a little data massaging) to do a basic replay of traffic found in our log files.

The process is broken up into two components: setting up the test host, and 
running the test.

All options for Under Siege can be specified either via command-line arguments for 
via .under_siege config files in either the current directory or your home directory.
Configuration is loaded first from the home directory, then the current directory,
then finally from command-line arguments, with each subsequent config set overriding
any configuration from the previous set.

Under Siege (`us_prepare_test_urls`) expects a log-file format that is something close
to the Apache combined format. It is specifically written to work with Varnish
NCSA log files. In particular, it does the following regular expression
search on each line in the log:

    /"GET (http:\/\/[^\s]+)/

and makes use of the `$1` match. For example, the following line:

    83.170.113.102 "-" [28/Feb/2012:16:08:18 -0500] "GET http://www.middlebury.edu/ HTTP/1.0" 200 0 "-" "pingdom" 0.000042915 hit

would result in a match of

    http://www.middlebury.edu/

If you log file has a different format, edit lines 97-99 of `us_prepare_test_urls` 
to fit your file format. Similarly, you may wish to edit `us_filter_urls` to exclude
certain URLs from your test set.

Installation
-------------

1. Clone the repository to your test client

        git clone git://github.com/adamfranco/under_siege.git /usr/local/src/under_siege
    
2. Add the Under Siege scripts to your path either by updating your path:

        export PATH=$PATH:/usr/local/src/under_siege
    
    Or, by making symbolic links to the scripts in your bin/ directory:
   
        cd /usr/local/bin
        ln -s /usr/local/src/under_siege/us_filter_urls
        ln -s /usr/local/src/under_siege/us_prepare_test_urls
        ln -s /usr/local/src/under_siege/us_run_test
        ln -s /usr/local/src/under_siege/us_unique_hosts
        ln -s /usr/local/src/under_siege/us_update_hosts_file
    
3. (Optional) Add a `.under_siege` configuration file to your home directory or 
   the current directory. System defaults can be specified in a file 
   at /etc/under_siege. See below for configuration options.

Setting up the test
-------------------

1. Copy your log file to your test client.
2. Run `us_prepare_test_urls` with your log files as input:
    
    us_prepare_test_urls /path/to/varnishncsa.log

If you want to just test a portion of the log file, you can filter it and pipe it
to `us_prepare_test_urls`:

    grep "20/Feb/2012" /path/to/varnishncsa.log | us_prepare_test_urls

`us_prepare_test_urls` will in turn call several scripts to filter the URLs and
add all of the hostnames in these URLs to the machine's /etc/hosts file so that
siege will target your test server rather than using your production host as found
via DNS.
  
### Command-line/Configuration options

    Usage:
        us_prepare_test_urls [-h] [-f testfile] [-i ip-address] [-y|-n] <log files>
        
        -h  Display help.
        -f  File path for test URL list output.
        -i  The IP address of the testing host to add to the /etc/hosts file.
        -y  Apply changes to /etc/hosts without prompting.
        -n  Skip changes to /etc/hosts without prompting.
        -r  Remove hosts that are no longer in the host list. Useful if your hosts
            file is getting bloated.
        
        <log files>  Varnish or Apache log files in something close to the Apache 
                     "combined" format.
    
    Configuration:
        A configuration file named .under_siege can be placed in the current directory
        or in your home directory. Command-line arguments can be used to override
        config entries.
        
        $TEST_FILE  (string) The file path where the test file will be output.
                             Example: $TEST_FILE = '/tmp/under_siege_test_urls.txt';
                             
        $IP         (string) The IP address of the test web host. Will be added to the
                             /etc/hosts file for each hostname in the test file.
                             Example: $IP = '140.233.4.221';
                             
        $YES_FOR_ALL (int)   If nil or 0, you will be prompted for all /etc/hosts changes.
                             If 1, all /etc/hosts changes will be applied without prompt.
                             If -1, all /etc/hosts changes will be skipped without prompt.

        $REMOVE_HOSTS (int)  If 0 (default), new hosts in the list will be added to the
                             /etc/hosts file, but existing entries will be left in place.
                             If 1, the hosts file will be pruned to just the hosts in the
                             list.

Running the test
----------------

Run `us_run_test` with any Siege options you would like:

    us_run_test -c 200

By default `us_run_test` will use the `$TEST_FILE` from your configuration, though
this can be overridden via the `-f` command line option (useful if you have multiple
test files you wish to alternate between).

### Command-line/Configuration options

    Usage:
        us_run_test [-h] [-f testfile] [-b] [-c CONCURRENT] [-C] [-r REPS] [-R RC] [-d DELAY]
        
        -h  Display this help.
        -f  File path for list of URLs to test.
        -b  BENCHMARK, signifies no delay for time testing.
        -c  CONCURRENT users, default is 10
        -C  CONFIGURATION, show the current siege configuration.
        -r  REPS, number of times to run the test, default is 25
        -R  RC, change the siegerc file to file.  Overrides
            the SIEGERC environmental variable.
        -d  Time DELAY, random delay between 1 and num designed
            to simulate human activity. Default value is 3
    
    Configuration:
        A configuration file named .under_siege can be placed in the current directory
        or in your home directory. Command-line arguments can be used to override
        config entries.
        
        $TEST_FILE  (string) The file path where the test file will be output.
                             Example: $TEST_FILE = '/tmp/under_siege_test_urls.txt';
        
        $SIEGE_ARGS (string) command-line arguments to pass to siege.
                             Example: $SIEGE_ARGS = '-c 25 -v';
