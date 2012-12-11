---
layout: post
title: "An iOS app for plotting live data: ConAir:iOS"
date: 2012-12-11 21:28
comments: true
categories: [objective-c, iOS]
---

In previous posts on this blog we've built a basic environmental monitoring system
which exposes data as a simple JSON webservice. This post is looking at how to
build an iOS app to consume the timeseries data. We'll establish the following:

* A datasource object which pulls data from a webservice
* Setting the data source to poll for new data
* Create a UI which updates when new data arrives
* Plotting the data in a chart

All of this can be applied to any webservice available, but since we have built
something suitable as part of this blog we'll use that.

<!-- more -->

## A datasource

We'll create a class which is responsible for retrieving the data, and storing
a local cache of it. Our different UI controllers will interact with this class
to get hold of datapoints to render.

We will only want one instance of a datasource at any one time - multiple data
sources in one application would be both memory intensive, and would cause multiple
network requests for the same data. We therefore make the datasource a singleton:

{% codeblock ConairDatasource.h lang:objc %}
#import <Foundation/Foundation.h>

@interface VPYConairDataSource : NSObject

@property (nonatomic, strong, readonly) NSMutableArray *data;

+ (VPYConairDataSource*)sharedDataSource;

@end
{% endcodeblock %}

This header also exposes a readonly data array - which we will use later. The
equivalent implementation is as follows:

{% codeblock ConairDatasource.m lang:objc %}
#import "ConairDataSource.h"

@interface VPYConairDataSource ()
@property (nonatomic, strong, readwrite) NSMutableArray *data;
@end

@implementation VPYConairDataSource

+ (VPYConairDataSource *)sharedDataSource
{
    static VPYConairDataSource *sharedDataSource = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedDataSource = [[self alloc] init];
    });
    return sharedDataSource;
}

@end
{% endcodeblock %}

We've redefined the readonly data property to be readwrite, and implemented the
`+sharedDataSource` class method, to create a singleton.

In the first instance we're going to pull the data from the webservice when the
datasource is created, and to that end, we implement the standard constructor, and
a utility method:

{% codeblock Pulling data from the webservice lang:objc %}
- (id)init
{
    self = [super init];
    if (self) {
        // Grab the inital data import
        [self collectDataFromInternet];
    }
    return self;
}

- (void)collectDataFromInternet
{
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    NSDate *startDate = [NSDate dateWithTimeIntervalSinceNow:-24*60*60];
    if(self.data && self.data.count != 0) {
        startDate = [[self.data lastObject] objectForKey:@"ts"];
    }
    NSDate *endDate = [NSDate date];
    
    NSString *urlString = [NSString stringWithFormat:@"http://sl-conair.herokuapp.com/data/?start=%@&stop=%@&step=%@",
                           [startDate dateAsISO8601String], [endDate dateAsISO8601String], @"120000"];
    NSURL *url = [NSURL URLWithString:urlString];
    dispatch_async(queue, ^{
        NSLog(@"requesting");
        NSData *data = [NSData dataWithContentsOfURL:url];
        
        // Parse the JSON data
        NSError *error;
        NSArray *json = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&error];
        if(error) {
            NSLog(@"There was an error");
        } else {
            // Convert the JSON structure into a nice array
            NSMutableArray *dataPoints = [[NSMutableArray alloc] init];
            NSDateFormatter *df = [[NSDateFormatter alloc] init];
            [df setTimeStyle:NSDateFormatterFullStyle];
            [df setDateFormat:@"yyyy-MM-dd HH:mm:ss Z"];
            NSDate *currentLatestDate = [[self.data lastObject] objectForKey:@"ts"];
            [json enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
                NSDate *ts = [df dateFromString:obj[@"ts"]];
                // Let's check that we have a new data point
                if(!currentLatestDate || !([ts isEqualToDate:currentLatestDate] || ([ts compare:currentLatestDate] == NSOrderedAscending))) {
                    NSDictionary *datapoint = @{@"ts" : ts, @"temperature" : obj[@"temperature"]};
                    [dataPoints addObject:datapoint];
                }
            }];
            
            if (dataPoints.count > 0) {
                // Perform the assignment on the main thread
                dispatch_async(dispatch_get_main_queue(), ^{
                    if(!self.data) {
                        self.data = dataPoints;
                    } else {
                        for (NSDictionary *dp in dataPoints) {
                            [self insertObject:dp inDataAtIndex:self.data.count];
                        }
                    }
                });
            }
        }
    });
}

{% endcodeblock %}

## Polling for new data


## A simple auto-updating UI


## Plotting data in a chart