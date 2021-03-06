* Example morphline file.
```
morphlines : [
  {
    id : morphline1
    # need all commands
    importCommands : ["org.kitesdk.**"]

    commands : [
      {
        readLine {
          charset : UTF-8
        }
      }
      #{ logInfo { format : "output record: {}", args : ["@{message}"] } }
      {
        tryRules {
          catchExceptions : true
          rules : [
            # first try
            {
              commands : [
                {
                  grok {
                    dictionaryResources : [grok-dictionaries/grok-patterns, grok-dictionaries/linux-syslog]
                    dictionaryString : """
                     SYSLOG5424NGNAME %{WORD:syslog5424_appname1}-%{WORD:syslog5424_appname2}
                     SYSLOG5424BASE1 %{SYSLOG5424PRI}%{NONNEGINT:syslog5424_ver} +(?:%{TIMESTAMP_ISO8601:syslog5424_ts}|-) +(?:%{HOSTNAME:syslog5424_host}|-) +(?:%{SYSLOG5424NGNAME:syslog5424_app}) +(?:%{WORD:syslog5424_proc}|-) +(?:%{WORD:syslog5424_msgid}|-) +(?:%{SYSLOG5424SD:syslog5424_sd}|-|)
                     SYSLOG5424LINE1 %{SYSLOG5424BASE1} +%{GREEDYDATA:syslog5424_msg}
                    """
                    expressions : {
                      message : """<%{POSINT:syslog_pri}>%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}"""
                    }
                  }
                }
                {
                  split {
                    inputField : syslog5424_ts
                    outputFields : [timestamp, timezone]
                    separator : "+"
                    isRegex : false
                  }
                }
                {
                  convertTimestamp {
                    field : timestamp
                    inputFormats : ["yyyy-MM-dd'T'HH:mm:ss", "MMM d HH:mm:ss"]
                    inputTimezone : Asia/Seoul
                    outputFormat : "unixTimeInMillis"
                    outputTimezone : UTC
                  }
                }
                {
                  setValues {
                    key : "@{syslog_hostname},@{syslog_program}"
                    value : "1"
                  }
                }
              ]
            }

            # oops. Exception Record.
            {
              commands : [
                {
                  setValues {
                    key : "@{exceptionkey}"
                    value : "@{message}"
                  }
                }
              ]
            }
          ]
        }
      }
    ]
  }
]
```
* tryRules
 * catchExceptions is set to true. So Exceptions can be handled another commands.
 * May insert another commands for second try between two commands set.
 * Use exceptionkey field for exception record. This key handled by ExceptionPartitioner.