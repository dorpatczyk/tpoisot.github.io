---
layout: post
title: Using networkx to simulate metapopulations in Python
summary: blog
chapo: Simulation of spatially explicit metapopulations in Python
author: Tim
type: note
tags:
- python
- networkx
- metapopulation
---

Python progressively replaced R for most of my daily simulations needs, because it's relatively fast, easy to write, and a pleasure to read. There are a lot of great packages for scientists, and since I started working on food webs seriously, I've been using `networkx` more and more. We've been having discussions in the lab about spatial graphs, surrounding a recent paper by [Gilarranz and Bascompte](http://www.ncbi.nlm.nih.gov/pubmed/22155351) and some of our own projects, and I thought that it will be fun to use `networkx` to run a super simple metapopulation simulation. So without further ado, the recipe! Try to follow through the code, or get the final version as a [gist](https://gist.github.com/2725839).

# Setting things up

Before we start, we'll need a few things. Some modules are required: obviously `networkx` to deal with the graph objects, `numpy` to generate random numbers, and the `pyplot` module to deal with the output. I'd much rather do the output work using `pyx`, but it would add a good 50 lines of code to get a good-looking output, so let's go the easy way.

{% highlight python %}
import networkx as nx
import numpy as np
import matplotlib.pyplot as plt
{% endhighlight %}

Then we'll set a few parameters:

{% highlight python %}
Patches = 100   # Number of patches
P_ext = 0.01    # Probability of extinction (e)
P_col = 0.014   # Probability of colonization (c)
P_init = 0.02   # Probability that a patch will be occupied at the beginning
Distance = 1.4  # An arbitrary parameter to determine which patches are connected
{% endhighlight %}

# Create a node class

The first thing to do is to create a class for the patches, which will be the nodes of our spatial graph. This is relatively easy to do, and we call this new class `patch`. Before writing up, let's think about what to put in. We need a simple parameter which we call `status`, whose value can be either `0` (the patch is empty) or `1` (the patch is occupied).

We'll also create a tuple called `pos`, to store the position of the node in a two-dimensional space. The resulting class is:

{% highlight python %}
class patch:
    def __init__(self,status=0,pos=(0,0)):
        self.status = status
        self.pos = pos
    def __str__(self):
        return(str(self.status))
{% endhighlight %}

I won't go into the detail of this notation, but you can read more [here](http://www.penzilla.net/tutorials/python/classes/) or [here](http://jhamrick.mit.edu/2011/05/18/an-introduction-to-classes-and-inheritance-in-python/) if you want. In any case, we now have a way to specify patches, with a spatial position and an occupancy. With this in hand, the next step is to create a network of patches, which will be the spatial landscape over which we simulate our metapopulation.

# Create a spatially explicit graph

There are a lot of ways to create spatial graphs, but for the sake of simplicity, let us assume that we will use simples rules. Nodes (patches) have a position (*x,y*), and two nodes are connected by an edge if the euclidean distance between them is inferior or equal to a fixed value. In other words, we can decide how close two patches should be to allow migration to occur.

In `networkx`, creating a graph is easy, and is done by

{% highlight python %}
G = nx.Graph()
{% endhighlight %}

We will now generate as many (`Patches`) patches as needed, and put them on the landscape at random.

{% highlight python %}
for i in xrange(Patches):
    Stat = 1 if np.random.uniform() < P_init else 0
    Pos  = (np.random.uniform()*10-5,np.random.uniform()*10-5)
    G.add_node(patch(Stat,Pos))
{% endhighlight %}

This require at least Python 2.6, but you can easily rewrite the problematic first line as an `if/else` block. Basically, this will generate random positions in space, and randomly decide if the patch is occupied or not. This is where `networkx` is extremely powerful: nodes can be any Python objects. In the simplest cases, you may want to use only numbers or strings, but if you have to keep track of the nodes in more advanced ways, then this become extremely fun and convenient to work with.

At this point, nodes are not connected by dispersal yet. To decide which patches are connected, we will loop through them all, and access their `pos` attributes:

{% highlight python %}
for p1 in G.nodes():
    for p2 in G.nodes():
        Dist = np.sqrt((p1.pos[1]-p2.pos[1])**2+(p1.pos[0]-p2.pos[0])**2)
        if Dist <= Distance:
            G.add_edge(p1,p2)
{% endhighlight %}

For all pairs of nodes, we check that their distance falls below the treshold for migration, and if so, we create an edge in the network.

At this point, it's easy to visualize the network:

{% highlight python %}
occup = [n.status for n in G]
pos = {}
for n in G.nodes():
    pos[n] = n.pos
nx.draw(G,node_color=occup,with_labels=False,cmap=plt.cm.Greys,pos=pos,vmin=0,vmax=1)
plt.show()
{% endhighlight %}

This will open a Python plotting view looking like:

![Figure1]({{ site.url }}/images/metapop_step1.png)

The black nodes are occupied patches, and the white ones are empty patches. If the landscape it too connected (or not connected enough), act on the `Distance` parameter to fix it.

# Start the simulation

So now, we have everything to start the simulation. We'll first do a loop to update the status of the nodes, starting with extinctions, and then colonization. Let's start with the extinction routine:

{% highlight python %}
for n in G.nodes():
    if (n.status == 1 and np.random.uniform() < P_ext):
        n.status = 0
{% endhighlight %}

Simply put, for each node, if it is occupied and a random extinction event occurs, then its status is reverted to 0. Super easy. We can do something similar for the colonization routine. We assume that each occupied node will be able to colonize a single of its neighbouring nodes. This rquires to get a list of nodes adjacent to the node currently being evaluated.

{% highlight python %}
for n in G.nodes():
    if n.status == 1:
        neighb = G[n] # That's it, a list of the neighbors
        for nei in neighb:
            if nei.status == 0:
                if np.random.uniform() < P_col:
                    nei.status = 1
                    break
{% endhighlight %}

That's a lot of loops within loops, but it goes rougly as follows: for each of the nodes, if their is a population, we check the neighbors. For the non-occupied neighbors, we check if the colonization occurs, and if so, we break the loop.

With these two blocks, it's easy to do the actual simulation:

{% highlight python %}
for timestep in xrange(2000):
    # Check for extinctions
    for n in G.nodes():
        if (n.status == 1 and np.random.uniform() < P_ext):
            n.status = 0
    # Check for invasions
    for n in G.nodes():
        if n.status == 1:
            neighb = G[n] # That's it, a list of the neighbors
            for nei in neighb:
                if nei.status == 0:
                    if np.random.uniform() < P_col:
                        nei.status = 1
                        break
{% endhighlight %}

We then plot the lanscape using the same command as before (I've actually added the list of occupancies directly within the plot instruction, because one-liners looks cool):

{% highlight python %}
nx.draw(G,node_color=[n.status for n in G],with_labels=False,cmap=plt.cm.Greys,pos=pos,vmin=0,vmax=1)
plt.show()
{% endhighlight %}

![Figure2]({{ site.url }}/images/metapop_final.png)

But of course you would like to follow the dynamics, which is extremely easy. We will just add, just before the beginning of the simulation loop, the lines

{% highlight python %}
Time = [0]
Occupancy = [np.sum([n.status for n in G])/float(Patches)]
{% endhighlight %}

And we will add infos at the end of each loop, i.e. immediately after the `break` instruction (but in the main loop, not the colonization one):

{% highlight python %}
Time.append(timestep+1)
Occupancy.append(np.sum([n.status for n in G])/float(Patches))
{% endhighlight %}

And at the really end of the simulation, just add

{% highlight python %}
plt.plot(Time,Occupancy,'g-')
plt.show()
{% endhighlight %}

Which will give you the following result:

![Figure3]({{ site.url }}/images/metapop_dynamics.png)

# Conclusions

And that's it! Using the ability of `networkx` to make any arbitrary object into a node, it's possible to implement a metapopulation simulation in roughly 50 lines. The package is reasonably fast, so it's possible to run more complex simulations (I've been playing with metacommunities with no problem, for example).
