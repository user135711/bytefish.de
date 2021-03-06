﻿title: Building Applications with Apache Flink (Part 5): Complex Event Processing with Apache Flink
date: 2016-07-10 16:10
tags: java, flink, cep
category: java
slug: apache_flink_series_5
author: Philipp Wagner
summary: This article shows how to work with the Complex Event Processing Engine of Apache Flink.

In this article I want to show you how to work with the Complex Event Processing (CEP) engine of Apache Flink. 

The Apache Flink documentation describes [FlinkCEP] as:

> FlinkCEP is the complex event processing library for Flink. It allows you to easily detect complex event patterns in a stream 
> of endless data. Complex events can then be constructed from matching sequences. This gives you the opportunity to quickly get 
> hold of what's really important in your data.

## What we are going to build ##

Imagine we are designing an application to generate warnings based on certain weather events.

The application should generate weather warnings from a Stream of incoming measurements:

* Extreme Cold (less than -46 °C for three days)
* Severe Heat (above 30 °C for two days)
* Excessive Heat (above 41°C for two days)
* High Wind (wind speed between 39 mph and 110 mph)
* Extreme Wind (wind speed above 110 mph)

## Source Code ##

You can find the full source code for the example in my git repository at:

* [https://github.com/bytefish/FlinkExperiments](https://github.com/bytefish/FlinkExperiments)

## Warnings and Patterns ##

First of all we are designing the model for Warnings and their associated patterns.

All warnings in the application derive from the marker Interface ``IWarning``.

```java
// Copyright (c) Philipp Wagner. All rights reserved.
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

package app.cep.model;

/**
 * Marker interface used for Warnings.
 */
public interface IWarning {

}
```

A warning is always generated by a certain pattern, so we create an interface ``IWarningPattern`` for it. The actual patterns for [Apache Flink] will be defined with the [Pattern API]. 

Once a pattern has been matched, Apache Flink emits a ``Map<String, TEventType>`` to the environment, which contains the names and events of the match. So implementations of 
the ``IWarningPattern`` also define how to map between the Apache Flink result and a certain warning. 

And finally to simplify reflection when building the Apache Flink stream processing pipeline, the ``IWarningPattern`` also returns the type of the warning.

```java
// Copyright (c) Philipp Wagner. All rights reserved.
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

package app.cep.model;

import org.apache.flink.cep.pattern.Pattern;

import java.io.Serializable;
import java.util.List;
import java.util.Map;

/**
 * A Warning Pattern describes the pattern of a Warning, which is triggered by an Event.
 *
 * @param <TEventType> Event Type
 * @param <TWarningType> Warning Type
 */
public interface IWarningPattern<TEventType, TWarningType extends IWarning> extends Serializable {

    /**
     * Implements the mapping between the pattern matching result and the warning.
     *
     * @param pattern Pattern, which has been matched by Apache Flink.
     * @return The warning created from the given match result.
     */
    TWarningType create(Map<String, List<TEventType>> pattern);

    /**
     * Implementes the Apache Flink CEP Event Pattern which triggers a warning.
     *
     * @return The Apache Flink CEP Pattern definition.
     */
    Pattern<TEventType, ?> getEventPattern();

    /**
     * Returns the Warning Class for simplifying reflection.
     *
     * @return Class Type of the Warning.
     */
    Class<TWarningType> getWarningTargetType();

}
```

### Excessive Heat Warning ###

Now we can implement the weather warnings and their patterns. The patterns are highly simplified in this article. 

One of the warnings could be a warning for Excessive Heat, which is described on Wikipedia as:

> Excessive Heat Warning – Extreme Heat Index (HI) values forecast to meet or exceed locally defined warning criteria for at least two days.
> Specific criteria varies among local Weather Forecast Offices, due to climate variability and the effect of excessive heat on the local
> population.
>
> Typical HI values are maximum daytime temperatures above 105 to 110 °F (41 to 43 °C) and minimum nighttime temperatures above 75 °F (24 °C).

#### Warning Model ####

The warning should be issued if we expect daytime temperatures above 41 °C for at least two days. So the ``ExcessiveHeatWarning`` class 
takes two ``LocalWeatherData`` measurements, and also provides a short summary in its ``toString`` method.

```java
// Copyright (c) Philipp Wagner. All rights reserved.
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

package app.cep.model.warnings.temperature;

import app.cep.model.IWarning;
import model.LocalWeatherData;

public class ExcessiveHeatWarning implements IWarning {

    private final LocalWeatherData localWeatherData0;
    private final LocalWeatherData localWeatherData1;

    public ExcessiveHeatWarning(LocalWeatherData localWeatherData0, LocalWeatherData localWeatherData1) {
        this.localWeatherData0 = localWeatherData0;
        this.localWeatherData1 = localWeatherData1;
    }

    public LocalWeatherData getLocalWeatherData0() {
        return localWeatherData0;
    }

    public LocalWeatherData getLocalWeatherData1() {
        return localWeatherData1;
    }

    @Override
    public String toString() {
        return String.format("ExcessiveHeatWarning (WBAN = %s, First Measurement = (%s), Second Measurement = (%s))",
                localWeatherData0.getStation().getWban(),
                getEventSummary(localWeatherData0),
                getEventSummary(localWeatherData1));
    }

    private String getEventSummary(LocalWeatherData localWeatherData) {

        return String.format("Date = %s, Time = %s, Temperature = %f",
                localWeatherData.getDate(), localWeatherData.getTime(), localWeatherData.getTemperature());
    }
}

```

#### Warning Pattern ####

Now comes the interesting part, the ``Pattern``. The ``ExcessiveHeatWarningPattern`` implements the ``IWarningPattern`` interface and 
uses the [Pattern API] to define the matching pattern. You can see, that we are using strict contiguity for the events, using the 
the ``next`` operator. The events should occur for the maximum temperature of 2 days, so we expect these events to be within 2 days.

```java
// Copyright (c) Philipp Wagner. All rights reserved.
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

package app.cep.model.patterns.temperature;

import app.cep.model.IWarningPattern;
import app.cep.model.warnings.temperature.ExcessiveHeatWarning;
import model.LocalWeatherData;
import org.apache.flink.cep.pattern.Pattern;
import org.apache.flink.cep.pattern.conditions.SimpleCondition;
import org.apache.flink.streaming.api.windowing.time.Time;

import java.util.List;
import java.util.Map;

/**
 * Excessive Heat Warning – Extreme Heat Index (HI) values forecast to meet or exceed locally defined warning criteria for at least two days.
 * Specific criteria varies among local Weather Forecast Offices, due to climate variability and the effect of excessive heat on the local
 * population.
 *
 * Typical HI values are maximum daytime temperatures above 105 to 110 °F (41 to 43 °C) and minimum nighttime temperatures above 75 °F (24 °C).
 */
public class ExcessiveHeatWarningPattern implements IWarningPattern<LocalWeatherData, ExcessiveHeatWarning> {

    public ExcessiveHeatWarningPattern() {}

    @Override
    public ExcessiveHeatWarning create(Map<String, List<LocalWeatherData>> pattern) {
        LocalWeatherData first = pattern.get("First Event").get(0);
        LocalWeatherData second = pattern.get("Second Event").get(0);

        return new ExcessiveHeatWarning(first, second);
    }

    @Override
    public Pattern<LocalWeatherData, ?> getEventPattern() {
        return Pattern
                .<LocalWeatherData>begin("First Event").where(
                        new SimpleCondition<LocalWeatherData>() {
                            @Override
                            public boolean filter(LocalWeatherData event) throws Exception {
                                return event.getTemperature() >= 41.0f;
                            }
                        })
                .next("Second Event").where(
                        new SimpleCondition<LocalWeatherData>() {
                            @Override
                            public boolean filter(LocalWeatherData event) throws Exception {
                                return event.getTemperature() >= 41.0f;
                            }
                        })
                .within(Time.days(2));
    }

    @Override
    public Class<ExcessiveHeatWarning> getWarningTargetType() {
        return ExcessiveHeatWarning.class;
    }
}
```

## Converting a Stream into a Stream of Warnings ##

Now it's time to apply these patterns on a ``DataStream<TEventType>``. In this example we are operating on the Stream of historical weather measurements, which 
have been used in previous articles. These historical values could easily be exchanged with forecasts, so it makes a nice example.

I have written a method ``toWarningStream``, which will take a ``DataStream<LocalWeatherData>`` and generate a ``DataStream`` with the warnings.

```java
// Copyright (c) Philipp Wagner. All rights reserved.
// Licensed under the MIT license. See LICENSE file in the project root for full license information.

package app.cep;

import app.cep.model.IWarning;
import app.cep.model.IWarningPattern;
import app.cep.model.patterns.temperature.SevereHeatWarningPattern;
import app.cep.model.warnings.temperature.SevereHeatWarning;
import model.LocalWeatherData;
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.api.java.typeutils.GenericTypeInfo;
import org.apache.flink.cep.CEP;
import org.apache.flink.cep.PatternSelectFunction;
import org.apache.flink.cep.PatternStream;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.AscendingTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import stream.sources.csv.LocalWeatherDataSourceFunction;
import utils.DateUtilities;

import java.time.ZoneOffset;
import java.util.Date;
import java.util.List;
import java.util.Map;

public class WeatherDataComplexEventProcessingExample {

    public static void main(String[] args) throws Exception {

        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // Use the Measurement Timestamp of the Event:
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // Path to read the CSV data from:
        final String csvStationDataFilePath = "C:\\Users\\philipp\\Downloads\\csv\\201503station.txt";
        final String csvLocalWeatherDataFilePath = "C:\\Users\\philipp\\Downloads\\csv\\201503hourly_sorted.txt";


        // Add the CSV Data Source and assign the Measurement Timestamp:
        DataStream<model.LocalWeatherData> localWeatherDataDataStream = env
                .addSource(new LocalWeatherDataSourceFunction(csvStationDataFilePath, csvLocalWeatherDataFilePath))
                .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<LocalWeatherData>() {
                    @Override
                    public long extractAscendingTimestamp(LocalWeatherData localWeatherData) {
                        Date measurementTime = DateUtilities.from(localWeatherData.getDate(), localWeatherData.getTime(), ZoneOffset.ofHours(0));

                        return measurementTime.getTime();
                    }
                });

        // First build a KeyedStream over the Data with LocalWeather:
        KeyedStream<LocalWeatherData, String> localWeatherDataByStation = localWeatherDataDataStream
                // Filter for Non-Null Temperature Values, because we might have missing data:
                .filter(new FilterFunction<LocalWeatherData>() {
                    @Override
                    public boolean filter(LocalWeatherData localWeatherData) throws Exception {
                        return localWeatherData.getTemperature() != null;
                    }
                })
                // Now create the keyed stream by the Station WBAN identifier:
                .keyBy(new KeySelector<LocalWeatherData, String>() {
                    @Override
                    public String getKey(LocalWeatherData localWeatherData) throws Exception {
                        return localWeatherData.getStation().getWban();
                    }
                });

        // Now take the Maximum Temperature per day from the KeyedStream:
        DataStream<LocalWeatherData> maxTemperaturePerDay =
                localWeatherDataByStation
                        // Use non-overlapping tumbling window with 1 day length:
                        .timeWindow(Time.days(1))
                        // And use the maximum temperature:
                        .maxBy("temperature");

        // Now apply the SevereHeatWarningPattern on the Stream:
        DataStream<SevereHeatWarning> warnings =  toWarningStream(maxTemperaturePerDay, new SevereHeatWarningPattern());

        // Print the warning to the Console for now:
        warnings.print();

       // Finally execute the Stream:
        env.execute("CEP Weather Warning Example");
    }

    private static <TWarningType extends IWarning> DataStream<TWarningType> toWarningStream(DataStream<LocalWeatherData> localWeatherDataDataStream, IWarningPattern<LocalWeatherData, TWarningType> warningPattern) {
        PatternStream<LocalWeatherData> tempPatternStream = CEP.pattern(
                localWeatherDataDataStream.keyBy(new KeySelector<LocalWeatherData, String>() {
                    @Override
                    public String getKey(LocalWeatherData localWeatherData) throws Exception {
                        return localWeatherData.getStation().getWban();
                    }
                }),
                warningPattern.getEventPattern());

        DataStream<TWarningType> warnings = tempPatternStream.select(new PatternSelectFunction<LocalWeatherData, TWarningType>() {
            @Override
            public TWarningType select(Map<String, List<LocalWeatherData>> map) throws Exception {
                return warningPattern.create(map);
            }
        }, new GenericTypeInfo<TWarningType>(warningPattern.getWarningTargetType()));

        return warnings;
    }

}
```

[Apache Flink]: https://flink.apache.org/
[FlinkCEP]: https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/libs/cep.html
[Pattern API]: https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/libs/cep.html#the-pattern-api
[DataStream]: https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/index.html
[KeyedStream]: https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/windows.html
[SourceFunction]: https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/#data-sources
[SinkFunction]: https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/#data-sinks
[PgBulkInsert]: https://github.com/bytefish/PgBulkInsert