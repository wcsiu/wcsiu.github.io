---
comments: true
layout: post
title:  "A better Response struct for InfluxDB Golang client"
date:   2018-05-31 10:31:38 +0800
tags:
  - InfluxDB
  - Golang
---

InfluxDB is great for reading real time IoT devices data and the write performace is superb 

I have been using [InfluxDB client][influxdb-client] with Golang for a while and the experience is by far very comfortable but I found the `Response` from InfluxDB client is very uneasy to use.

The `Response` returned by query

{% highlight go %}
//client/v2/client.go
type Response struct {
    Results []Result
    Err     string `json:"error,omitempty"`
}

type Result struct {
    Series   []models.Row
    Messages []*Message
    Err      string `json:"error,omitempty"`
}
{% endhighlight %}

{% highlight go %}
//models/rows.go
type Row struct {
    Name    string            `json:"name,omitempty"`
    Tags    map[string]string `json:"tags,omitempty"`
    Columns []string          `json:"columns,omitempty"`
    Values  [][]interface{}   `json:"values,omitempty"`
    Partial bool              `json:"partial,omitempty"`
}
{% endhighlight %}

The main problem is the `Row` struct design. Basically everytime you want to get a value from the data you retrieved, you have to loop through `Columns` in `Row` to retrieve the corresponding index of the field you are looking for in `Values`. It is not impossible to use but the `Row` struct design will make your code looks ugly because of the tedious loops.

Here is a code snippet I use to make the `Row` easier to use.

{% highlight go %}
func RowToMap(row models.Row) map[string][]interface{} {
	ret := make(map[string][]interface{})
	for i, v := range row.Columns {
		ret[v] = make([]interface{}, len(row.Values))
		for k, values := range row.Values {
			ret[v][k] = values[i]
		}
	}
	return ret
}
{% endhighlight %}

Now we can easily loop through our query result with simple key, value for-loop or merge two `Row`s with `append` if we want.

Feel free to share your thoughts by commenting below.

[influxdb-client]: https://github.com/influxdata/influxdb/tree/master/client