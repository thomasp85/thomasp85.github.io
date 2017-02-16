---
title: "Introduction to ggraph: Edges"
description: "In the third post in this series of ggraph introductions I will dive into how to connect nodes using edges"
tags: [R, ggraph, visualization]
categories: [R]
large_thumb: true
img:
    thumb: "/assets/images/edge_meme.jpg"
---



This is the third post in my series of `ggraph` introductions. The 
[first]({% post_url 2017-02-06-ggraph-introduction-layouts %}) post introduced
the concept of layouts, which is simply a specification on how nodes should be
placed on a plane. The [second]({% post_url 2017-02-10-ggraph-introduction-nodes %})
explained how to draw nodes using the `geom_node_*()` family. This post will 
connect the dots, so to speak, by introducing the concept of edges.

## Edges
If the natural `ggplot2` equivalent to nodes is `geom_point()`, then surely the
equivalent to edges must be `geom_segment()`? Well, sort of, but there's a bit 
more to it than that.

![One does not simply draw a line between two nodes](/assets/images/edge_meme.jpg)

While nodes are the sensible, mature, and predictably geoms, edges are the edgy 
(sorry), younger cousins that pushes the boundaries. To put it bluntly: 

> On the ggraph savannah you definetly want to be an edge!

### Meet the `geom_edge_*()` family
While the introduction might feel a bit over-the-top it is entirely true. An 
edge is an abstract concept denoting a relationship between two enteties. A 
straight line is simply just one of many ways this relationship can be 
visualised. As we saw when [discussing nodes]({% post_url 2017-02-10-ggraph-introduction-nodes %})
sometimes it is not drawn at all but impied using containment or position 
(treemap, circle packing, and partition layouts), but more often it is shown
using a line of some sort. This use-case is handled by the large family of edge
geoms provided in `ggraph`. Some of the edges are general while others are 
dedicated to specific layouts. Lets creates some graphs for illustrary purpose
first:


{% highlight r %}
library(ggraph)
library(igraph)
hierarchy <- as.dendrogram(hclust(dist(iris[, 1:4])))

# Classify nodes based on agreement between children
hierarchy <- tree_apply(hierarchy, function(node, children, ...) {
    if (is.leaf(node)) {
        attr(node, 'Class') <- as.character(iris[as.integer(attr(node, 'label')),5])
    } else {
        classes <- unique(sapply(children, attr, which = 'Class'))
        if (length(classes) == 1 && !anyNA(classes)) {
            attr(node, 'Class') <- classes
        } else {
            attr(node, 'Class') <- NA
        }
    }
    attr(node, 'nodePar') <- list(class = attr(node, 'Class'))
    node
}, direction = 'up')

hairball <- graph_from_data_frame(highschool)

# Classify nodes based on popularity gain
pop1957 <- degree(delete_edges(hairball, which(E(hairball)$year == 1957)), 
                  mode = 'in')
pop1958 <- degree(delete_edges(hairball, which(E(hairball)$year == 1958)), 
                  mode = 'in')
V(hairball)$pop_devel <- ifelse(pop1957 < pop1958, 'increased',
                                ifelse(pop1957 > pop1958, 'decreased', 
                                       'unchanged'))
V(hairball)$popularity <- pmax(pop1957, pop1958)
E(hairball)$year <- as.character(E(hairball)$year)
{% endhighlight %}

#### Link
While you don't have to use a straight line for edges it is certainly possible
and `geom_edge_link()` is here to serve your needs:


{% highlight r %}
ggraph(hairball, layout = 'kk') + 
    geom_edge_link(aes(colour = year))
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-3-1.png)

There's really not much more to it --- every edge is simply a straight line 
between the terminal nodes. Moving on...

#### Fan
Sometimes the graph is not simple, i.e. it has multiple edges between the same
nodes. Using links is a bad choice here because edges will overlap and the
viewer will be unable to discover parallel edges. `geom_edge_fan()` got you 
covered here. If there are no parallel edges it behaves like `geom_edge_link()`
and draws a straight line, but if parallel edges exists it will spread them out
as arcs with different curvature. Parallel edges will be sorted by 
directionality prior to plotting so edges flowing in the same direction will be
plottet together:


{% highlight r %}
ggraph(hairball, layout = 'kk') + 
    geom_edge_fan(aes(colour = year))
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-4-1.png)

#### Loops
Loops cannot be shown with regular edges as they have no length. A dedicated 
`geom_edge_loop()` exists for these cases:


{% highlight r %}
# let's make some of the student love themselves
loopy_hairball <- add_edges(hairball, rep(1:5, each=2), year = rep('1957', 5))
ggraph(loopy_hairball, layout = 'kk') + 
    geom_edge_link(aes(colour = year), alpha = 0.25) + 
    geom_edge_loop(aes(colour = year))
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-5-1.png)

The direction, span, and strength of the loop can all be controlled, but in 
general loops will add a lot of visual clutter to your plot unless the graph is
very simple.

#### Density
This one is definetly strange, and I'm unsure of it's usefulness, but it is here
and it deserves an introduction. Consider the case where it is of interest to
see which types of edges dominates certain areas of the graph. You can colour
the edges, but edges can tend to get overplotted, thus reducing readability.
`geom_edge_density()` lets you add a shading to your plot based on the density
of edges in a certain area:


{% highlight r %}
ggraph(hairball, layout = 'kk') + 
    geom_edge_density(aes(fill = year)) + 
    geom_edge_link(alpha = 0.25)
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-6-1.png)

#### Arcs
While some insists that curved edges should be used in standard *"hairball"* 
graph visualisations it really is a poor choice, as it increases overplotting
and interpretability for virtually no gain (unless complexiness is your thing).
That doesn't mean arcs have no use in graph visualizations. Linear and circular
layouts can benefit greatly from them and `geom_edge_arc()` is provided 
precisely for this scenario:


{% highlight r %}
ggraph(hairball, layout = 'linear') + 
    geom_edge_arc(aes(colour = year))
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-7-1.png)

Arcs behave differently in circular layouts as they will always bend towards
the center no matter the direction of the edge (the same thing can be achieved
in a linear layout by setting `fold = TRUE`).


{% highlight r %}
ggraph(hairball, layout = 'linear', circular = TRUE) + 
    geom_edge_arc(aes(colour = year)) + 
    coord_fixed()
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-8-1.png)

#### Elbow
Aah... The classic dendrogram with its right angle bends. Of course such
visualizations are also supported with the `geom_edge_elbow()`. It goes without
saying that this type of edge requires a layout that flows in a defined 
direction, such as a dendrogram:


{% highlight r %}
ggraph(hierarchy, layout = 'dendrogram') + 
    geom_edge_elbow()
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-9-1.png)

#### Diagonals
If right angles aren't really your thing `ggraph` provides a smoother version in
the form of `geom_edge_diagonal()`. This edge is a quadratic bezier with control
points positioned at the same x-value as the terminal nodes and halfway 
in-between the nodes on the y-axis. The result is more organic than the elbows:


{% highlight r %}
ggraph(hierarchy, layout = 'dendrogram') + 
    geom_edge_diagonal()
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-10-1.png)

It tend to look a bit weird with hugely unbalanced trees so use with care...

#### Hive
This is certainly a very specific type of edge, intended only for use with hive
plots. It draws edges as quadratic beziers with control point positioned 
perpendicular to the axes of the hive layout:


{% highlight r %}
ggraph(hairball, layout = 'hive', axis = 'pop_devel', sort.by = 'popularity') + 
    geom_edge_hive(aes(colour = year)) + 
    geom_axis_hive() + 
    coord_fixed()
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-11-1.png)

## The three types of edge geoms
Almost all edge geoms comes in three variants. The basic variant (no suffix) as 
well as the variant suffixed with 2 (e.g. `geom_edge_link2()`) calculates a 
number (`n`) of points along the edge and draws it as a path. The variant 
suffixed with 0 (e.g. `geom_edge_diagonal0()`) uses the build in grid grobs to
draw the edges directly (in case of a diagonal it uses `bezierGrob()`). It might
seem strange to have so many different implementations of the same geoms but 
there's a reason to the insanity...

### Base variant
The basic edge geom is drawn by calculating a number of points along the edge
path and draw a line between these. This means that you're in control of the
detail level of curved edges and that all complex calculations happens up front.
Generally you will see better performance using the base variant rather than the
0-variant that uses grid grobs, unless you set the number of points to calculate
to something huge (50--100 is usually sufficient for a smooth look). Apart from
better performance you also get a nice bonus (you actually get several, but only
one is discussed here): The possibility of drawing a gradient along the edge.
Each calculated point gets an index value between 0 and 1 that specifies how
far along the edge it is positioned and this value can be used to e.g. map to
an alpha level to show the direction of the edge:


{% highlight r %}
ggraph(hairball, layout = 'linear') + 
    geom_edge_arc(aes(colour = year, alpha = ..index..)) + 
    scale_edge_alpha('Edge direction', guide = 'edge_direction')
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-12-1.png)

### 2-variant
Like the base variant the 2-variant calculates points along the edge and draws
a path along them. The difference here is that in this variant you can map node
attributes to the edge and the aesthetics are then interpolated along the edge.
This is easier to show than to explain:


{% highlight r %}
ggraph(hierarchy, layout = 'dendrogram') + 
    geom_edge_elbow2(aes(colour = node.class))
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-13-1.png)

There are considerably more computation going on than in the base variant so 
unless you need to interpolate values between the terminal nodes you should go
with the base variant.

### 0-variant
This is, sadly, the boring one at the end. You don't get the luxury of smooth
gradients over the edge and often you have a considerably worse performance.
What you gain though is tack sharp resolution in the curves so if this is of 
utmost importance you are covered by this variant.

## Decorating edges
An edge is so much more than a line... Well at least it is also potentially an
arrow and a label. This section will go into how these can be added. To clearly
see the effect here we will use a slightly simpler graph


{% highlight r %}
simple <- make_graph('bull')
# Random names - I swear
V(simple)$name <- c('Thomas', 'Bob', 'Hadley', 'Winston', 'Baptiste')
E(simple)$type <- sample(c('friend', 'foe'), 5, TRUE)
{% endhighlight %}


### Arrows
While we saw above that direction can be encoded as a gradient, the good old 
arrow is still available. As with the standard `ggplot2` geoms an arrow can be 
added using the arrow argument:


{% highlight r %}
ggraph(simple, layout = 'graphopt') + 
    geom_edge_link(arrow = arrow(length = unit(4, 'mm'))) + 
    geom_node_point(size = 5)
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-15-1.png)

I hope you think *Ugh* at the sight of this. The edges naturally extend to the
node center and nodes are thus drawn on top of the arrow heads. There's a 
solution to this in the form of the `start_cap` and `end_cap` aesthetics in the
base and 2-variant edge geoms (sorry 0-variant). This can be used to start and
stop the edge drawing at an absolut distance from the terminal nodes. Wath this:


{% highlight r %}
ggraph(simple, layout = 'graphopt') + 
    geom_edge_link(arrow = arrow(length = unit(4, 'mm')), 
                   end_cap = circle(3, 'mm')) + 
    geom_node_point(size = 5)
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-16-1.png)

Using the `circle()`, `square()`, `ellipsis()`, and `rectangel()` helpers it is
possible to get a lot of control over how edges are capped at either end. This 
works for any edge, curved or not:


{% highlight r %}
ggraph(simple, layout = 'linear', circular = TRUE) + 
    geom_edge_arc(arrow = arrow(length = unit(4, 'mm')), 
                  start_cap = circle(3, 'mm'),
                  end_cap = circle(3, 'mm')) + 
    geom_node_point(size = 5) + 
    coord_fixed()
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-17-1.png)

When plotting node labels you often want to avoid that incoming and outgoing 
edges overlaps with the labels. `ggraph` provides a helper that calculates the 
bounding rectangle of the labels and cap edges based on that:


{% highlight r %}
ggraph(simple, layout = 'graphopt') + 
    geom_edge_link(aes(start_cap = label_rect(node1.name),
                       end_cap = label_rect(node2.name)), 
                   arrow = arrow(length = unit(4, 'mm'))) + 
    geom_node_text(aes(label = name))
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-18-1.png)

The capping of edges is dynamic and responds to resizing of the plot so the 
absolute size of the cap areas are maintained at all time.

#### A quick note on directionality
In `ggraph` there is no such thing as an undirected graph. Every edge has a 
start and an end node. For undirected graphs the start and end of edges is 
arbitrary but still exists and it is thus possible to add arrowheads to 
undirected graphs as well. This should not be done of course, but this is the
responsibility of the user as `ggraph` does not make any checks during 
rendering.

### Labels
You would expect that edge labels would be their own geom(s), but `ggraph` 
departs from the stringent grammar interpretation here. This is because the 
label placement is dependent on the choice of edge. Because of this edge 
labeling is bundled with each edge geom (but not the 0-variant) through the 
label aesthetic


{% highlight r %}
ggraph(simple, layout = 'graphopt') + 
    geom_edge_link(aes(label = type), 
                   arrow = arrow(length = unit(4, 'mm')), 
                   end_cap = circle(3, 'mm')) + 
    geom_node_point(size = 5)
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-19-1.png)

Usually you would like the labels to run along the edges, but providing a 
fixed angle will only work at a very specific aspect ratio. Instead `ggraph`
offers to calculate the correct angle dynamically so the labels always runs
along the edge. Furthermore it can offset the label by an absolute length:


{% highlight r %}
ggraph(simple, layout = 'graphopt') + 
    geom_edge_link(aes(label = type), 
                   angle_calc = 'along',
                   label_dodge = unit(2.5, 'mm'),
                   arrow = arrow(length = unit(4, 'mm')), 
                   end_cap = circle(3, 'mm')) + 
    geom_node_point(size = 5)
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-20-1.png)

`ggraph` offers a lot of additional customization of the edge labels but this
shows the main features. As with arrowheads labels can severely clutter your
visualization so it is only advisable on very simple graphs.

## Connections
The estranged cousin of edges are connections. While edges show the relational 
nature of the nodes in the graph structure, connections connect nodes that are 
not connected in the graph. This is done by finding the shortest path between
the two nodes. Currently the only connection geom available is 
`geom_conn_bundle()` that implements the hierarchical edge bundling techinque:


{% highlight r %}
flaregraph <- graph_from_data_frame(flare$edges, vertices = flare$vertices)
from <- match(flare$imports$from, flare$vertices$name)
to <- match(flare$imports$to, flare$vertices$name)
ggraph(flaregraph, layout = 'dendrogram', circular = TRUE) + 
    geom_conn_bundle(data = get_con(from = from, to = to), alpha = 0.1) + 
    coord_fixed()
{% endhighlight %}

![center](/assets/images/2017-02-16-ggraph-introduction-edges/unnamed-chunk-21-1.png)

The connection concept is underutilized at the moment but I expect to add more
support for this in comming releases.

## The end
This concludes my tour of the main features of `ggraph` --- I hope you are all
well prepared to take your network visualizations to the next level!
