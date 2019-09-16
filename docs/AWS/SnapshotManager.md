# Snapshot Manager Note

## Overview of the problem we want to solve

In dailly AWS operations, we need to make snapshot and copy them to another region for backup purpose.  
This is not a new problem and there are many existing solutions.  

The interesting part is how can we keep the snapshots efficiently.  
For example, if those snapshots were made every 30 minutes, you will have 48 snapshots in a day and 480 snapshots in 10 days.  

In previous solutions, they just simply setup a policy to remove those old snapshots by a fixed number or date.  
The problem is if you set to keep 30 snapshots, it will only keep less than a days backup and if you set to keep 10 days, you end up wasting a lot of storage to keep these 480 snapshots.  

A much better solution is to have an "Elastic" rule to remove those snapshots based on their create time.  
For example, we can set the rule like this:

* If this snapshot was made in recent 5 hours, then we keep it 
* If it's between 5h to 12h, we keep only 1 per hour
* If it's from 12h to 24h, we only keep one every 4 hours
* If it's after 2days, we only keep 1 per day
* If it's over 14 days, we don't keep

This solution is like the RRDTools db algorithm, we keep the data based on their freshness. Older data will be treated differently than newer data.  
This algorithm is what we want to implement here.

## Simplify the problem first

Lets try to solve a simplified version first then we can use the knowledge to implement a much complete solution.  

First, we set the data we want to process to a simple array(LIST) with integer from 1 to 50
```
Data = [i+1  for i in range(50)]
```

Then we define our rule to be like this
```
Rule = {
    "5": 3,
    "10": 6,
    "25": 10,
    "40": -1
}
```
This rule means:

- If data >= 5, keep one every 3 
- If data >=10, keep one every 6
- If data >=25, keep one every 10
- If data >=40, don't keep

Then we sort the Rule's keys 
```
ruleKeys = sorted([ int(v) for v in rule.keys() ])
```
now `ruleKeys` is like [5, 10, 25, 40]

for any given data as `d`, we will check which range it fits in the `ruleKeys`

Here we use `bisect.bisect_left` to do a quick bisect search
```
def find_lt(a, x):
    'Find rightmost value less than x'
    i = bisect_left(a, x)
    if i:
        return a[i-1]
    raise ValueError
```

For example, if data is 6
`k = find_lt(ruleKeys, 6)` will give you the `rightmost value in ruleKeys less than 6` which is 5  

Now we know 6 is in the range of 5 to 10, and based on the `Rule` dict, we know in that range, we want to keep one every 3
```
"5": 3
```
We call this 3 `interval`
```
interval = rule[str(k)]
```

Then we can calculate which "bucket" our data 6 will fit in
```
bucketNumb = (d - k) // interval
```
For example, `(6 - 5) // 3` will give 0 and `(8 - 5) // 3` will give 1  
That means 5, 6, 7 will have same bucket number 0 and 8, 9, 10 will be in bucket 1 and 11, 12, 13 will be in bucket 2  
We can just use the bucket number to decide whether we need to keep this data or not  
Say we have 5 then there is no need to keep 6 and 7  
if we have 12, there is no need to keep 13 because they are in same bucket.

So the ProcessData looks like this
```
def ProcessData(data, rule):
    ruleKeys = sorted([ int(v) for v in rule.keys() ])
    lastBucket = -1
    lastRuleKey = -1
    for d in data:
        ifRemove = False
        try:
            k = find_le(ruleKeys, d)
            interval = rule[str(k)]
            if interval <= 0:
                ifRemove = True
            else:
                if k == d:
                    lastBucket = 0
                    lastRuleKey = k
                else:
                    bucketNumb = (d - k) // interval
                    if k != lastRuleKey:
                        lastRuleKey = k
                        lastBucket = bucketNumb
                    else:                    
                        if bucketNumb == lastBucket:
                            ifRemove = True
                        else:
                            lastBucket = bucketNumb
        except ValueError:
            # k is less than ruleKeys[0], simply pass so it won't be REMOVED
            pass
        if ifRemove:
            print("{0:3d}  {1:10s}".format(d, "REMOVE"))
        else:
            print("{0:3d}  {1:10s}".format(d, "OK"))
```


## Next step, use this algorithm in the snapshot management program

First, we need to set the Rule based on time difference of current time and the creation time of the snapshots

```
# current time in UTC
utc_now = datetime.now(timezone.utc)

# time difference in seconds between "now" and the snapshots create time
int((utc_now - s['StartTime']).total_seconds())
```

Next, we need to figure out a way to specify the Rules more intuitively

We'd like the Rule to be readable, something like this:
```
Rule = {
    "10h": "20m",
    "11h10m": 1800,
    "12h": "40m",
    "1d": "1d"
    "2d": -1
}
```

In the example above, "10h" means 10 hours, 20m means 20 minutes, 1d means 1 day

Here comes a handy utility function to convert these "time string" to regular timedelta

```
regex = re.compile(r'((?P<days>\d+?)d)?((?P<hours>\d+?)h)?((?P<minutes>\d+?)m)?((?P<seconds>\d+?)s)?')

def timestr_to_seconds(time_str):
    # check if time_str is a numeric value like 1200 or -1
    time_str = str(time_str)
    if re.match(r"^\d+$", time_str):
        # if it's just an integer, assume it's seconds
        return int(time_str)
    # otherwise try to parse the value using RegEx
    parts = regex.search(time_str)
    if not parts:
        return None
    parts = parts.groupdict()
    time_params = {}
    for (name, param) in parts.items():
        if param:
            time_params[name] = int(param)
    return timedelta(**time_params).total_seconds()
```

Finally, we can combine all these to a more complete solution like:

```
def process_snapshots(rule):
    # Get my AWS Account ID
    myAccount = boto3.client('sts').get_caller_identity()['Account']

    # Connect to EC2
    client = boto3.client('ec2', region_name = 'us-west-2')

    # Get a list of snapshots for my AWS account (not all public ones)
    snapshots = client.describe_snapshots(OwnerIds=[myAccount])['Snapshots']
    print(snapshots)

    # utc_origin = datetime.utcfromtimestamp(0) # this gives 1970.1.1
    # utc_now = datetime.utcnow()
    utc_now = datetime.now(timezone.utc)
    # now_seconds = (utc_now - utc_origin).total_seconds()
    # print(utc_origin, utc_now, now_seconds)

    snapshots_seconds = [ 
        {
            "sec": int((utc_now - s['StartTime']).total_seconds()),
            "ind": i
        }
            for i, s in enumerate(snapshots) 
    ]
    # sort this list by the "sec" value, so it will be in asc order
    snapshots_seconds = sorted(snapshots_seconds, key=lambda x: x['sec'])
    print(snapshots_seconds)

    # process snapshots based on rules
    newRule = {}
    for k,v in rule.items():
        rule_key = int(utils.timestr_to_seconds(k))
        rule_val = int(utils.timestr_to_seconds(v))
        newRule[str(rule_key)] = rule_val

    ruleKeys = sorted([ int(v) for v in newRule.keys() ])
    rule = newRule
    lastBucket = -1
    lastRuleKey = -1
    for v in snapshots_seconds:
        d = v['sec']
        i = v['ind']
        ifRemove = False
        try:
            k = find_le(ruleKeys, d)
            interval = rule[str(k)]
            if interval <= 0:
                ifRemove = True
            else:
                if k == d:
                    lastBucket = 0
                    lastRuleKey = k
                else:
                    bucketNumb = (d - k) // interval
                    if k != lastRuleKey:
                        lastRuleKey = k
                        lastBucket = bucketNumb
                    else:
                        if bucketNumb == lastBucket:
                            ifRemove = True
                        else:
                            lastBucket = bucketNumb
        except ValueError:
            # k is less than ruleKeys[0], simply pass so it won't be REMOVED
            pass
        if ifRemove:
            print("{} sec:{} SnapshotId:{} {:10s}".format(i, d, snapshots[i]['SnapshotId'], "REMOVE")) 
        else:
            print("{} sec:{} SnapshotId:{} {:10s}".format(i, d, snapshots[i]['SnapshotId'], "OK"))
```


