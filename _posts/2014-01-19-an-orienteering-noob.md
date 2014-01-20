---
layout: post
title: Placeholders in Apache Camel routes
tags: [Apache Camel, Java]
thumbnail: 
synopsis: Spring property placeholders don't work natively in Apache Camel routes so you have to find workarounds to be able to softcode values in your routes. This post addresses how to do this for the Aggregate EIP.
---
One of the nice things that Spring allows you to do is to add property placeholders to your configuration files and then define the values at runtime. The options are endless as to what you might want to use this feature for. One thing you cannot do though is to use a spring property placeholder inside Camel XML. You can find more details about why here http://camel.apache.org/how-do-i-use-spring-property-placeholder-with-camel-xml.html. Looking specifically at aggregators, imagine you wanted to do this:
 
    <route id="routeName">
        <from ref="inbound"/>
        <aggregate strategyRef="aggregationStrategy" completionSize="${property.placeholder}">
            <correlationExpression>
                <constant>VALUE</constant>
            </correlationExpression>
        </aggregate>
    </route>

Well, you can't. The ${property.placeholder} will not be substituted with the appropriate value.
 
There are a number of ways around this issue, but one I rather like goes something like this.
 
First, change the aggregator to use a completionSize element and set the value to a header value:
 
    <route id="routeName">
        <from ref="inbound"/>
        <aggregate strategyRef="aggregationStrategy">
            <correlationExpression>
                <constant>VALUE</constant>
            </correlationExpression>
            <completionSize>
                <header> COMPLETION_SIZE</header>
            </completionSize>
        </aggregate>
    </route>

This means that the header property COMPLETION_SIZE will be used to determine the completion size. So how do you get that header value set. Like this:
 
    <route id="routeName">
        <from ref="inbound"/>
        <bean ref="addAggregationParameters"/>
        <aggregate strategyRef="aggregationStrategy">
            <correlationExpression>
                <constant>VALUE</constant>
            </correlationExpression>
            <completionSize>
                <header> COMPLETION_SIZE</header>
            </completionSize>
        </aggregate>
    </route>
 
And this is what the bean looks like:
 
    <bean id="addAggregationParameters" class="com.nomura.fitp.referencedata.listener.AddHeaders">
        <constructor-arg name="headers">
            <map>
                <entry key="COMPLETION_SIZE" value="${property.placeholder}"/>
            </map>
        </constructor-arg>
    </bean>
 
That class is really simple and looks like this:
 
    public final class AddHeaders
    {
        private final Map<String, Object> headers;

        public AddHeaders(Map<String, Object> headers)
        {
            this.headers = headers;
        }

        public void addHeaders(Exchange exchange)
        {
            Message message = exchange.getIn();
            for (Map.Entry<String, Object> header : headers.entrySet()) {
                message.setHeader(header.getKey(), header.getValue());
            }
        }
    }
 
And that's it. 