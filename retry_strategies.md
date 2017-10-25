# Robust backlog processing

In this article we will present two simple solutions ("Fibonacci retry"
and "Progressive schedule") for a common pattern in backend programs: A
database table consisting of records that are to be processed by a
periodically running task. Here is an example of such a table (let's
call it BACKLOG):

    +-----+---------+------------------+
    | ID | STATUS   |  DATE INSERTED   |
    +-----+---------+------------------+
    |  4  | NEW     | ( 8 minutes ago) |
    |  3  | NEW     | ( 5   hours ago) |
    |  2  | NEW     | ( 3    days ago) |
    |  1  | NEW     | ( 2  months ago) |
    +-----+---------+------------------+

And an example of a processing function:

    def process_backlog():
        try:
            sql = '''
                select *
                from BACKLOG
                where status in ('NEW', 'ERROR')
                and   date_inserted >= ?
                order by date_inserted desc
                limit ?
            '''
            for record in execute(
                sql, [now - config.max_age, config.max_records]
            ):
                do_actual_processing(record)
        except:
            record.status = 'ERROR'
        else:
            record.status = 'SUCCESS'
        finally:
            record.save()

Using our favorite scheduler, say cron, we set process_backlog() to run
every M minutes. After one call to process_backlog() we might have the
following situation:

    +-----+---------+------------------+
    | ID | STATUS   |  DATE INSERTED   |
    +-----+---------+------------------+
    |  4  | ERROR   | ( 8 minutes ago) |
    |  3  | SUCCESS | ( 5   hours ago) |
    |  2  | SUCCESS | ( 3    days ago) |
    |  1  | ERROR   | ( 2  months ago) |
    +-----+---------+------------------+

During its next run process_backlog() will retry records 1 and 4. It
will keep doing so every M minutes until they can be successfully
processed. There are some problems with this approach:

    * There might be something inherently wrong with this record that
      will prevent it from being processed correctly for the foreseeable
      future or forever. If enough such records occur then they will
      prevent any useful work from being done.
    * do_actual_processing() might be calling an external service that
      we don't want to call every M minutes, especially since this
      service appears to have a temporary(?) problem.

To avoid these problems we need to process pending BACKLOG records less
and less frequently as time goes on.

## Fibonacci retry

The Fibonacci sequence is 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, ... . Using
such a retry strategy means that a record with 1 or 2 failures will be
retried after 1 minute (or days or whatever), but a record with 6
failures after 8 minutes (or days or whatever).  But to employ this
strategy we will need two extra columns in the BACKLOG table:

    +-----+---------+------------------+-----------------------------+
    | ID | STATUS   |  DATE INSERTED   | FAILURES | LAST_FAILURE     |
    +-----+---------+------------------+-----------------------------+
    |  4  | ERROR   | ( 8 minutes ago) |     1    | (2 minutes ago)  |
    |  3  | SUCCESS | ( 5   hours ago) |     0    | (2 minutes ago)  |
    |  2  | SUCCESS | ( 3    days ago) |     1    | (2 minutes ago)  |
    |  1  | ERROR   | ( 2  months ago) |     0    | (2 minutes ago)  |
    +-----+---------+------------------+-------+---------------------+

And the process_backlog() function (still scheduled to run every M
minutes) would be modified like so:

     def process_backlog():
         try:
             sql = '''
                 select *
                 from BACKLOG
                 where status in ('NEW', 'ERROR')
                 and   date_inserted >= ?
                 order by date_inserted desc
                 limit ?
             '''
             for record in execute(
                 sql, [now - config.max_age, config.max_records]
             ):
    +            fib_value = fibonacci(record.failures)
    +            if (now - record.last_failure < fib_value * 60):
    +                continue
                 do_actual_processing(record)
         except:
             record.status = 'ERROR'
    +        record.failures += 1
    +        record.last_failure = now
         else:
             record.status = 'SUCCESS'
         finally:
             record.save()

This way we avoid both of the aforementioned problems. But is there a
way to achieve the same without the need for the extra two database
columns?

## Progressive schedule

In this strategy we retain the original table, without added columns:

    +-----+---------+------------------+
    | ID | STATUS   |  DATE INSERTED   |
    +-----+---------+------------------+
    |  4  | ERROR   | ( 8 minutes ago) |
    |  3  | SUCCESS | ( 5   hours ago) |
    |  2  | SUCCESS | ( 3    days ago) |
    |  1  | ERROR   | ( 2  months ago) |
    +-----+---------+------------------+

But instead of the previous simple schedule of "run process_backlog()
every M minutes" we use the following "progressive" schedule:

    * Run process_backlog(  1) every   5 minutes
    * Run process_backlog(  7) every   1    hour
    * Run process_backlog( 14) every  12   hours
    * Run process_backlog( 30) every  24   hours
    * Run process_backlog(180) every  96   hours
    * Run process_backlog(360) every 192   hours

We have introduced an argument to the process_backlog() function which
refers to the maximum age of the records that it will try:

    +def process_backlog(ndays):
    -def process_backlog():
         try:
             sql = '''
                 select *
                 from BACKLOG
                 where status in ('NEW', 'ERROR')
                 and   date_inserted >= ?
                 order by date_inserted desc
                 limit ?
             '''
             for record in execute(
    +            sql, [now - ndays * 3600 * 24, config.max_records]
    -            sql, [now - config.max_age, config.max_records]
             ):
                 do_actual_processing(record)
         except:
             record.status = 'ERROR'
         else:
             record.status = 'SUCCESS'
         finally:
             record.save()

Thus, each record will be tried:

    * Once every 5 minutes for 1 day, then
    * Once every 1 hour for 1 week, then
    * Once every 12 hours for 2 weeks, then
    * Once every 24 hours for 1 month, etc.

## Conclusion

Retrying failed backlog records is more involved than it sounds. But
using a frequency inverse to the age of the record is simple and avoids
the problems of simple constant-frequency retry.
