---
layout: post_page
title:  "Trees in MySql - Breadcrumb"
date:   2014-05-11 22:04:06
categories: mysql
---

We continue from Adjacency Lists. We shall continue to try and model a set of comments with the structure shown below:


<pre>
            1
          /   \
         2     3
        /       \
       4         5
     /   \
    6     7
</pre>

The breadcrumb model is conceptually similar to the adjacency list model, except that instead of each node storing its direct parent, each node stores its entire list of ancestors up to its root. This can be useful if you need to get the full hierarchy of a node in a single query, especially when you don't know the depth of the node. If we were to represent the tree shown above, it would look like this:

<pre>
+------+------------+
| id   | breadcrumb |
+------+------------+
|    1 |          1 |
|    2 |        1/2 |
|    3 |        1/3 |
|    4 |      1/2/4 |
|    5 |      1/3/5 |
|    6 |    1/2/4/6 |
|    7 |    1/2/4/7 |
+------+------------+
</pre>

The delimiter used to show the hierarchy is up to you, I have chosen '/' here. 

### Pros

This model is much better at dealing with hierarchies of variable or unknown depth as compared to the adjacency list. This model is very useful when you have to get a list of all the parents to verify certain constraints as you can get that list in a single query.

### Cons

When you have to update the position of a element in the hierarchy, it you will also have to update all the children of that element. Also, you cannot easily work with the breadcrumb string within MySQL as it does not provide you with an easy string splitting function.

### From Adjacency List

You can create a breadcrumb model from an adjacency list(w depth listed) like so:

{% highlight sql %}

CREATE PROCEDURE generateBreadCrumbFromAdjacencyList()
BEGIN
  UPDATE comments SET breadcrumb = NULL;
  UPDATE comments SET breadcrumb = id WHERE depth = 1;
  SET @depth = 2;
  SET @id = (SELECT comments.id FROM comments WHERE depth = @depth AND breadcrumb IS NULL ORDER BY comments.id LIMIT 1);
  REPEAT
    IF((SELECT COUNT(1) FROM comments WHERE depth = @depth AND breadcrumb IS NULL) > 0) THEN
      SET @crumb = (SELECT CONCAT_WS('/', p.breadcrumb, u.id) FROM comments u, comments p WHERE u.parent = p.id AND u.id = @id);
      UPDATE comments u1 SET u1.breadcrumb = @crumb WHERE u1.id = @id;
      SET @id = (SELECT comments.id FROM comments WHERE depth = @depth AND breadcrumb IS NULL ORDER BY comments.id LIMIT 1);
    ELSE
      SET @depth = @depth + 1;
      SET @id = (SELECT comments.id FROM comments WHERE depth = @depth ORDER BY comments.id LIMIT 1);
    END IF;
  UNTIL (SELECT COUNT(1) FROM comments WHERE breadcrumb IS NULL) = 0 END REPEAT;
END;

{% endhighlight %}

You can attach this to triggers just like adjacency list to regenerate the breadcrumb column on update.

