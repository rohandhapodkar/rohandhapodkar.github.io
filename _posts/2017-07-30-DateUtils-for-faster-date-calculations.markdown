---
layout: post
title:  "DateUtils for faster date calculations"
date:   2017-07-30 12:56:04 +0530
categories: Java Date calculation
---
For past many years I used similar Date `truncate()` or `add()` operations as implemented in apache-commons-langs `DateUtils` class.

Have you ever investigated how many CPU cycles are wasted during every date manipulation ?

Refer to [DateCalculationsPerformanceTest](https://github.com/rohandhapodkar/suggested-java-best-practices/blob/master/src/test/java/test/java/util/date/DateCalculationsPerformanceTest.java) where `DateUtils.truncate()` method was called 10 Million times in many parallel threads.

| Scenario | Time (sec) |
|:-------|-------:|
| Add long numbers 10 M times | 0.046 |
| Call `org.apache.commons.lang3.time.DateUtils.truncate()` 10 M times | 18.967 |
| New `DateUtils.truncatedate()` 10 M times | 3.438 |


<br>

If your application is doing frequent date manipulations like truncate, addDate, get Next/Previous business date, then it's worth caching those results for better performance.

Below is the sample DateUtils implementation using ConcurrentSkipListMap for efficient date look up. This DateUtils class pre computes results for Date operations like nextDate.

This is just a prototype which can be extended further as per your requirements. eg.

- Caching first/last day of the month, adding some pre-configured days like SLA timelines from given date.
- Eager initialization of dates map.
- Return List/Iterator for all dates between given dates and so on.
To cache frequest Date to String conversion.

This prototype assumes 3 MB memory is not a constrain for your system and you are looking for performance efficient solution for frequent date calculations.
VisualVM shows retain size as 3 MB for all dates from 1970-1-1 to 2017-07-26.

- What are the limitations of this implementation

It assumes all your dates are using Single timezone. If you are looking for date calculations across multiple time zones then this implementation may not suite your requirement. 

{% highlight java linenos %}
package test.java.util.date;

import java.util.Calendar;
import java.util.Date;
import java.util.Map.Entry;
import java.util.concurrent.ConcurrentSkipListMap;

public class DateUtils {
	private static ConcurrentSkipListMap<Long, DateCalculations> dateMap = new ConcurrentSkipListMap<>();
	
	static class DateCalculations {
		
		Date keyDate;
		Date nextDate;
		
		public DateCalculations(Date inputDate) {
			keyDate = org.apache.commons.lang3.time.DateUtils.truncate(inputDate, Calendar.DATE);			
			nextDate = org.apache.commons.lang3.time.DateUtils.addDays(keyDate, 1);			
		}
		
		protected boolean coversDate(Date date) {
			long longDate = date.getTime();
			return this.keyDate.getTime() <= longDate && longDate < this.nextDate.getTime();
		}

		@Override
		public String toString() {
			return "DateCalculations [keyDate=" + keyDate + ", nextDate=" + nextDate + "]";
		}		
	}
	
	protected static int getMapSize() {
		return dateMap.size();
	}
	
	private static DateCalculations getDateCalculations(Date date) {
		Entry<Long, DateCalculations> existingEntry = dateMap.floorEntry(date.getTime());
		if(existingEntry != null && existingEntry.getValue().coversDate(date)) {
			return existingEntry.getValue();
		}
		DateCalculations newDateCalculation = new DateCalculations(date);
		
		assert newDateCalculation.coversDate(date) :" Failed ";
		
		dateMap.putIfAbsent(newDateCalculation.keyDate.getTime(), newDateCalculation);
		
		return getDateCalculations(date);
	}
	
	public static Date truncateDate(Date date) {
		return getWrappedDate(getDateCalculations(date).keyDate);
	}
	
	public static Date getNextDate(Date date) {
		return getWrappedDate(getDateCalculations(date).nextDate);
	}
	
	protected static Date getWrappedDate(Date date) {
		return (Date) date.clone();
	}
	
	public static void clearCache() {
		dateMap.clear();
	}
}

{% endhighlight %}
