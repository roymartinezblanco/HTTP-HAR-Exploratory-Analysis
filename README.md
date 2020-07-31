# HTTP-HAR-Exploratory-Analysis
 
TLDR; How to read a HAR file and make charts out of the curated data. 

---

A while back I wrote an article that talks about how to analyze an HTTP Archive (HAR), this blog is similar to it but I want to share how we can also get a lot of analytics from them. This is especially useful when no RUM (like mPulse) data is available.

[Past Blog: Har Analysis](https://roymartinez.dev/2019-06-05-har-analysis/)

What am I doing differently? This time I used Python libraries to provide multiple charts that can be of extreme use to us when consulting.

Libraries used (Pandas):

![Pandas](/Content/pandas.jpg)

```python
import json, os,re
import pandas as pd
import  matplotlib.pyplot as plt
```

I preferred to run this on a Jupiter Notebook, for simpler use and management but you can run this as a script  if you want. (see this for a quick guide if you want to do this too [Jupyter Guide](https://jupyter.org/install)).

> https://github.com/roymartinezblanco/HTTP-HAR-Exploratory-Analysis/

![Cache Status](/Content/cache-status.png)

## WHY?
My use for this is, to quickly execute it and get a good idea of what is happening and where to focus/start looking. One example is the following chart, using browser timing data. On it, we see that a couple of domains could benefit from browser hints ([Adaptive Acceleration](https://developer.akamai.com/ion/adaptive-acceleration)) to reduce DNS time along with other of the timings shown.

![3rd Party Connect Timing](/Content/3rd-connect-timings.png)

## First, start by cleaning the data.

This type of export from the browser contains ALOT of data and not all are useful. The idea is to have a dataset like the one shown below that can then be easily worked on.
![Sample Output](/Content/output.png)

I did this with two main snippets.

First, a common helper function to find and extract header values. this function takes a couple of values:

* **req** = JSON dataset containing details for the request in har >> log >> entries .
* **headerType** = request vs response, it indicates where to look.
* **headerName** = header to find
* **op** = currently, I've only put 2 (in and eq) but more can be added.

```python
def findHeader(req,headertype,headername,op = None):
    value = "None"
    if headertype == 'response':
        for h in req['response']['headers']:
            if op == 'in':
                if headername in h['name']:
                    value = h['value']
                    break
            else:
                if headername == h['name']:
                    value = h['value']
                    
                    break
    if headertype == 'cdn-timing':
        value = 0
        for h in req['response']['headers']:
            if op == 'eq':
                if 'server-timing' in h['name']:
                    if headername in h['value']:
                        
                        value = int(h['value'].split(';')[1].split('=')[1])
                        break
        if value is None:
            return 0
    return value
```

The following snippet is where we parse, clean, and filter the data in the har. In summary, it extracts that from headers to create the output CSV.

```python
colmms = ['url','host','host-type','method','status','ext','cpcode','ttl','server','cdn-cache','cdn-cache-parent','cdn-cache-key','cdn-req-id','vary','appOrigin','content-length','content-length-origin','blocked','dns','ssl','connect','send','ttfb','receive','edgeTime','originTime'
]
dat_clean = pd.DataFrame(columns=colmms)
for r in har['log']['entries']:
    u = str(r['request']['url']).split('?')[0]
    host = re.search('://(.+?)/', u, re.IGNORECASE).group(0).replace(':','').replace('/','')
    
    cachekey = str(findHeader(r,'response','x-cache-key','eq'))
    if not cachekey == 'None':
        cachekey = cachekey.split('/')
        cpcode = int(cachekey[3])
        ttl = cachekey[4]
        cdnCache = str(findHeader(r,'response','x-cache','eq')).split(' ')[0]
        cdnCacheParent = str(findHeader(r,'response','x-cache-remote','eq')).split(' ')[0]
        origin = str(findHeader(r,'response','x-cache-key','eq')).split('/')[5]
    else:
        cachekey = "None"
        cpcode = "None"
        ttl = "None"
        cdnCache = "None"
        cdnCacheParent = "None"
        origin = "None"

    ext = re.search(r'(\.[A-Za-z0-9]+$)', u, re.IGNORECASE)
    if any(tld in host for tld in FirstParty):
        hostType = 'First Party'
    else:
        hostType = 'Third Party'
    
    if ext is None:
        ext = "None"
    else:
        ext = ext.group(0).replace('.','') 
    ct = findHeader(r,'response','content-length','eq')
    if ct == "None":
        ct = 0
    else:
        ct = int(ct)
    if ext in ['jpg','png']:
        ct_origin = findHeader(r,'response','x-im-original-size','eq')
    else:
        ct_origin = findHeader(r,'response','x-akamai-ro-origin-size','eq')
    if ct_origin == "None":
        ct_origin = 0
    else:
        ct_origin = int(ct_origin)
    new_row = {
        'url':u,
        'host':host,
        'host-type':hostType,
        'method':r['request']['method'],
        'status':r['response']['status'],
        'ext':ext,
        'cpcode':cpcode,
        'ttl':ttl,
        'server':str(findHeader(r,'response','server','eq')),
        'cdn-cache':cdnCache,
        'cdn-cache-parent':cdnCacheParent,
        'cdn-cache-key':str(findHeader(r,'response','x-true-cache-key','eq')),
        'cdn-req-id':str(findHeader(r,'response','x-akamai-request-id','eq')),
        'vary':str(findHeader(r,'response','vary','eq')),
        'appOrigin':origin,
        'content-length':ct,
        'content-length-origin':ct_origin,
        'blocked':r['timings']['blocked'],
        'dns':r['timings']['dns'],
        'ssl':r['timings']['ssl'],
        'connect':r['timings']['connect'],
        'send':r['timings']['send'],
        'ttfb':r['timings']['wait'],
        'receive':r['timings']['receive'],
        'edgeTime':findHeader(r,'cdn-timing','edge','eq'),
        'originTime':findHeader(r,'cdn-timing','origin','eq') 
        }

    dat_clean = dat_clean.append(new_row,ignore_index=True)
dat_clean = dat_clean.groupby(colmms).size().reset_index(name='Count')   
dat_clean.to_csv(directory+'Output/output.csv',index=False)
```

## Examples

Here are some of the analysis that can be done with this setup and code snippet for each one.

### General

#### 3rd Party Timings

![3rd Party Timings](/Content/3rd-connect-timings.png) 

```python
tmp = dat_clean
tmp = tmp[tmp['host-type'] == 'Third Party']
tmp = tmp[["host", "send","ttfb", "receive"]]
tmp = tmp.groupby('host')[ "send", "ttfb", "receive"].mean().reset_index()
tmp.plot(x="host", kind="bar", stacked=True,label='Series')
plt.title('Third Party Timings')
plt.xlabel('Domains')
plt.ylabel('Milliseconds')
plt.show()
del tmp
```

#### First vs Third

![First vs Third](/Content/pie_first_vs_third.png)  

```python
tmp = dat_clean
tmp = tmp.groupby(['host','host-type','url']).size().reset_index(name='Count')
tmp = tmp.groupby(['host-type']).sum().reset_index()
tmp = tmp.sort_values(by=['Count'],ascending=False)
plt.pie(tmp['Count'],labels=tmp['host-type'],shadow=False,autopct='%1.1f%%')
plt.title('First vs Third Party')
plt.axis('equal')
plt.tight_layout()
plt.show()
```

#### 3rd Connect Timings

![3rd Connect Timings](/Content/3rd-connect-timings.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host-type'] == 'Third Party']
tmp = tmp.groupby('host')[ "dns","ssl","connect"].mean().reset_index()
tmp[["host", "dns","ssl","connect"]].plot(x="host", kind="bar", stacked=True,label='Series')
plt.title('Third Party Connect Timings')
plt.xlabel('Domains')
plt.ylabel('Milliseconds')
plt.show()
del tmp
```

#### HTTP Method by Domain

![HTTP Method by Domain](/Content/method_vs_domain.png)  

```python
tmp = dat_clean
tmp = tmp[["host","method"]]
tmp = tmp.groupby(['host','method']).size().reset_index(name='Count') 
tmp = tmp.reset_index().pivot(columns='method', index='host', values='Count')
tmp.plot.bar();
plt.title('Method by Domain')
plt.xlabel('Domains')
plt.ylabel('Count')
plt.show()
del tmp
```

#### Third Party by Requests

![Third Party by Requests](/Content/3rd_count.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host-type'] == 'Third Party']
tmp = tmp.groupby('host').size().reset_index(name='Count').plot(x="host", kind="bar", stacked=True,label='Series')
plt.title('Third Party by Requests')
plt.xlabel('Domain')
plt.ylabel('Requests')
plt.show()
del tmp
```

#### Third Party by Content Size

![Third Party by Content Size](/Content/3d_size.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host-type'] == 'Third Party']
tmp = tmp.groupby('host')[ "content-length"].mean().reset_index()
tmp[["host", "content-length"]].plot(x="host", kind="bar", stacked=True,label='Series')
plt.title('Third Party by Content Size')
plt.xlabel('Domains')
plt.ylabel('Content Length')
plt.show()
del tmp
```

#### HTTP Response Codes

![HTTP Response Codes](/Content/status_codes.png)  


```python
tmp = dat_clean 
tmp = tmp.groupby(['status']).size().reset_index(name='Count')
tmp = tmp.sort_values(by=['Count'],ascending=False)
plt.pie(tmp['Count'],labels=tmp['status'],shadow=False,autopct='%1.1f%%')
plt.title('HTTP Response Codes')
plt.axis('equal')
plt.tight_layout()
plt.show()
```

#### HTTP Status by Domain

![HTTP Status by Domain](/Content/status_by_domain.png)  

```python
tmp = dat_clean
tmp = tmp.groupby(['host','status']).size().reset_index(name='Count') 
tmp = tmp.reset_index().pivot(columns='status', index='host', values='Count')
tmp.plot.bar();
plt.title('HTTP Status by Domain')
plt.xlabel('Count')
plt.ylabel('Domain')
plt.show()
del tmp
```

#### HTTP Status by Domain

![HTTP Status by Domain](/Content/status_by_domain.png)  

```python
tmp = dat_clean
tmp = tmp.groupby(['host','status']).size().reset_index(name='Count') 
tmp = tmp.reset_index().pivot(columns='status', index='host', values='Count')
tmp.plot.bar();
plt.title('HTTP Status by Domain')
plt.xlabel('Count')
plt.ylabel('Domain')
plt.show()
del tmp
```

#### Extensions by Domain

![Extensions by Domain](/Content/ext_by_domain.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host-type'] == 'First Party']
tmp = tmp.groupby(['host','ext']).size().reset_index(name='Count') 
tmp = tmp.reset_index().pivot(columns='ext', index='host', values='Count')
tmp.plot.bar();
plt.title('Extensions by Domain')
plt.xlabel('Domain/Ext')
plt.ylabel('Count')
plt.show()
del tmp 
```

#### Timing by Ext

![Timing by Ext](/Content/timing_by_ext.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host'] == domain]
tmp = tmp.groupby('ext')[ "send", "ttfb", "receive"].mean().reset_index()
tmp[["ext", "send", "ttfb", "receive"]].plot(x="ext", kind="bar", stacked=True,label='Series')
plt.title(domain+': Timing by Ext')
plt.xlabel('Extensions')
plt.ylabel('Milliseconds')
plt.show()
del tmp  
```

### Akamai Specific

#### RO vs Origin

![RO vs Origin](/Content/ro_vs_or.png)  

```python
tmp = dat_clean
tmp = tmp[tmp['host-type'] == 'First Party']
tmp = tmp[tmp['ext'].isin(['css','js'])]
tmp = tmp[["ext", "content-length","content-length-origin"]]
tmp = tmp.groupby('ext')["content-length","content-length-origin"].mean().reset_index()
tmp.plot(x="ext", kind="bar",label=['RO','RAW'])
plt.title('First Party Resource Optimizer content-length vs Origin')
plt.xlabel('Extensions')
plt.ylabel('Bytes')
plt.legend(["RO", "Origin"]);
plt.show()
del tmp
```

#### IM vs Origin

![IM vs Origin](/Content/im_vs_or.png)  

```python
tmp = dat_clean
tmp = tmp[tmp['host-type'] == 'First Party']
tmp = tmp[tmp['ext'].isin(['jpg','png'])]
tmp = tmp.groupby('ext')["content-length","content-length-origin"].mean().reset_index()
tmp[["ext", "content-length","content-length-origin"]].plot(x="ext", kind="bar",label=['RO','RAW'])
plt.title('First Party Image Manager content-length vs Origin')
plt.xlabel('Extensions')
plt.ylabel('Bytes')
plt.legend(["IM", "Origin"]);
plt.show()
del tmp
```

#### Cache Status

![Cache Status](/Content/cache_status.png)  

```python
tmp = dat_clean 
tmp =  tmp[tmp['host'] == domain]
tmp = tmp.groupby(['cdn-cache']).size().reset_index(name='Count')
plt.pie(tmp['Count'],shadow=False,autopct='%1.1f%%')
plt.legend( tmp['cdn-cache'], loc="best")
plt.title('Cache Status')
plt.axis('equal')
plt.tight_layout()
plt.show()
del tmp  
```

#### Cache Status by ext

![Cache Status by ext](/Content/cache_status_by_ext.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host'] == domain]
tmp = tmp.groupby(['cdn-cache','ext']).size().reset_index(name='Count') 
tmp = tmp.reset_index().pivot(columns='ext', index='cdn-cache', values='Count')
tmp.plot.bar();
plt.title('Cache Status by ext')
plt.xlabel('Domain/Ext')
plt.ylabel('Count')
plt.show()
del tmp 
```

#### Edge vs Origin time by Ext

![Edge vs Origin time by Ext](/Content/edge_vs_or.png)  

```python
tmp = dat_clean
tmp =  tmp[tmp['host'] == domain]
tmp = tmp.groupby('ext')[ "edgeTime", "originTime"].mean().reset_index()
tmp[["ext", "edgeTime", "originTime"]].plot(x="ext", kind="bar", stacked=True,label='Series')
plt.title(domain+': Edge vs Origin time by Ext')
plt.xlabel('Extensions')
plt.ylabel('Milliseconds')
plt.show()
del tmp  
```


## Have fun!...