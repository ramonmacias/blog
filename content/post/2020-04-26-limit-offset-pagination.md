---
layout: post
title: "Limit/Offset"
subtitle: "An implementation of limit/offset pagination in Go"
date: 2020-04-26
author: "Ramon"
URL: "/2020/04/26/limit-offset-pagination/"
image: "img/playa_de_las_cuevas.jpg"
tags:
    - Go
    - Pagination
---

> The computer was born to solve problems that did not exist before. -- Bill Gates

## Pagination

One of the most common feature that will be face someday as a backend engineer, is how to deal with pagination on the server side. There is a different ways for implement that, and as you can imagine, each one has it a good and bad points. On this point I will try to implement the most basic one in Go, but also the idea is to implement the other different solutions.

So what's exactly the limit/offset approach? Most of the approaches about pagination ends with a query with the LIMIT and OFFSET values at the end of it, so the value on the limit, as the name says, is a limitation on the numbers of results you will receive from the query, for example if you want to have pages with a size of 10, the limit variable should take the value of 10. The offset value is basically the start point for the specific query, so for example the first page will take the offset value of 0, and the second page, if we take into account the page size if 10 will be as well 10 for the offset var, 20 for the third page and so on until the end.

This technique is easy to implement but presents two big problems. Result **inconsistency** and **offset inefficiency**. The inconsistency on this technique appears because we are not working each time with the same dataset, so take the following example, let's say we did a first call and received all the data of the first page, let's says 10 items, so we print these items and we ask for the second page, but as you can see, there is a gap of time between we call and print the first page and the second page call. So let's imagine we insert a new item in our dataset, that will be shown on the first page, this will lead us to an inconsistency because we will have again on the second page the last item of the first page, and we will miss this new item on that case. The offset inefficiency comes by the shifting the results by a large offset.

Another big issue that presents this solution, is that you have no idea about how many items you will have, this simplifies the implementation due we don't need to do an extra count query but, you lose the feature of moving to a specific page number or print how many pages you will have. But the point is, it worth the time of coding this approach and also to have to deal with the extra counter query? Sometimes yes but most of the times the customer doesn't care and most of the times they only want to be able to search and also sort the list.

## Implementation

I implemented a solution for that problem that you can check [here](https://github.com/ramonmacias/go-pagination/tree/master/limit-offset). On this repository I will try to post some other approaches, feel free to take the code an adde into our own pagination package.

All this package relies on the JSON API specification (https://jsonapi.org/format/#fetching-pagination) that defines the format of the parameters to use (page[limit] and page[offset]), as well the format of the json payload with the definition of the links object.

The pagination package also allow us to sort the query, for this case I didn't follow 100% the JSON API specification (https://jsonapi.org/format/#fetching-sorting), the specification says we should format the sort parameter in the next way. The sort order for each field should be ascending unless it is prefixed with a minus, that means for example

```
GET /articles?sort=-created,title
```

This endpoint will order by desc the created field and asc the title field, I decided to use the format of **field_name.order_sort**, so the above sample will be

```
GET /articles?sort=created.desc,title.asc
```

Why I decided to apply this approach? I think one of the key values when you are writing code is that should be legible, which means that with a quick look I should be able to understand what's going to happen, so in my opinion the second approach is more legible on the other hand we use more characters than we needed for do the same but desc and asc makes more sense than minus or plus.

This pagination as I said at the begining works using the approach of limit and offset, which means we avoid to have an extra count query each time we want to use the pagination engine. So how we deal with the last page issue? The answer is simple, if we have a limit of 10 that means I want to have pages with a size of 10 items, I will do a query of limit+1 and then I will check if we have some more items on the next page in order to know if I'm querying the last page or not, also then we deal with the removal of the extra item when we answer back, so the frontend still will receive always 10 items max instead of having the extra item requested.

In order to deal with that we should receive on the Paginate function a []interface{} and here is when some bad things appear, how we deal with the fact or article = interface{} is valid but []article = []interface{} is not valid. The standard golang recomendations told us how to do it https://golang.org/doc/faq#convert_slice_of_interface

So most of the time you should do something ugly like this

```
t := []int{1, 2, 3, 4}
s := make([]interface{}, len(t))
for i, v := range t {
    s[i] = v
}
```

Is not what I like but is the way for dealing on the management of removing last item data as well to check pagination links using len([]interface{}). Probably with the approach of having the extra query we will don't need to deal with this conversation.

Currently (19 April 2020), there is a proposal about generics in Go, that probably will change completely how we handle this []interface{}, I tried to implement a new version using the pre build go compiler with generics, but is to limited. The idea in general is to migrate into something that you can see on that article https://blog.tempus-ex.com/generics-in-go-how-they-work-and-how-to-play-with-them/

## How to use

The package will take the values from three different params;

* page[limit]
* page[offset]
* sort_by

So taking into account you will have some handler similar to this

```
func handler(wr http.ResponseWriter, req *http.Request) {

}
```

You will need to use the next function for get the pagination info

```
func handler(wr http.ResponseWriter, req *http.Request) {
  params, err := pagination.FindParams(req, defaultOffset, defaultLimit)

  params.Limit
  params.Offset
  params.Sort
}
```

Once you get the params, the next step is to inform to your function (that will retrieve all the data) about this params object, this params object has a method named Query() string, that will return a formatted query in order to be append on the end of your query, in the next snippet I will show you a case using sqlx

```
func handler(wr http.ResponseWriter, req *http.Request) {
  params, err := pagination.FindParams(req, defaultOffset, defaultLimit)

  data := getSomeCoolData(params)
}

func getSomeCoolData(params pagination.Params) []interface{} {
  data := []interface{}
  db.Select(
    &data,
    `SELECT * FROM cool_table`+params.Query,
  )
  return data  
}
```

This Query() method will append something like this **LIMIT 10 OFFSET 0 ORDER BY created_at desc**

The next step will be how to deal with the data result and paginate it, after you make your paginated query and get the results the last thing you have to do is to call the Paginate function.

```
func handler(wr http.ResponseWriter, req *http.Request) {
  params, err := pagination.FindParams(req, defaultOffset, defaultLimit)

  data := getSomeCoolData(params)

  json.NewEncoder(wr).Encode(pagination.Paginate(
    data,
    req.URL.EscapedPath(),
    params,
  ))
}
```

This will give an output like this

```
{
  "data": [
    {
      "custom_field": "awesome value"
    },
    //.....
  ],
  "links": {
    "first": "/data?page[limit]=10&page[offset]=0",
    "prev": "....",
    "next": "....",
    "last": "...."
  }
}
```
