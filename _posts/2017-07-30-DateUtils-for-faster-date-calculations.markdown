---
layout: post
title:  "DateUtils for faster date calculations"
date:   2017-07-30 12:56:04 +0530
categories: Java Date calculation
---
For many years I used similar Date `truncate()` or `add()` operations as implemented in apache-commons-langs `DateUtils` class.

Have you ever looked at how many CPU cycles we are wasting during every date manipulation ?

Refer to [DateCalculationsPerformanceTest](https://github.com/rohandhapodkar/suggested-java-best-practices/blob/master/src/test/java/test/java/util/date/DateCalculationsPerformanceTest.java) where `DateUtils.truncate()` method was called 10 Million times in 2 parallel threads.

| Scenario | Time (sec) |
|:-------|-------:|
| Add long numbers 10 M times | 0.046 |
| Call `org.apache.commons.lang3.time.DateUtils.truncate()` 10 M times | 18.967 |
| New `DateUtils.truncatedate()` 10 M times | 3.438 |

<br>
*Results based on Intel Core 2 Duo 2 GHz, Fedora 22 64-bit, Oracle JDK 1.8.0_131*


If your application is doing frequent date manipulations like truncate, addDate, get Next/Previous business date, then it's worth caching those results for better performance.

Below is the sample [`DateUtils`](#DateUtils.source) implementation using `ConcurrentSkipListMap` for efficient date look up. This DateUtils class pre computes results for Date operations like nextDate.

This is sample a prototype which can be extended further as per your requirements.

{{ site.data.format.list }} Some extension to this can be:

- Caching first/last day of the month, adding some pre-configured days for SLA.
- Eager initialization of dates map.
- Return List/Iterator for all dates between given dates.
- Cache Date to String conversion.

This prototype assumes few extra MB's does not overload your system in exchange of faster date calculations.

> VisualVM shows retain size as 3 MB for all dates between 1970-1-1 and 2017-07-26.


{{ site.data.format.list }} Why only `ConcurrentSkipListMap` ?

- It's a ConcurrentMap implementation.
- It's a `NavigableMap` map, which means `floorEntry()` method can be used to returns greatest date less than or equal to given Date. This is the best way to truncate the time from Date and lookup for corresponding Date.

{{ site.data.format.list }} What are the limitations of this implementation ?

- It assumes all your dates are using Single timezone. If you are looking for date calculations across multiple time zones then this implementation may not suit your requirement. 
- Not tested for dates prior to 1970-01-01


<a name="DateUtils.source"/>
{% highlight java %}
package test.java.util.date;

import java.util.Calendar;
import java.util.Date;
import java.util.Map.Entry;
import java.util.concurrent.ConcurrentSkipListMap;

public class DateUtils 
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
