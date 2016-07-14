---
layout: post
title:  Reynolds distance
date:   2016-07-12
categories: notes
---

A pdf of equation for various measures of genetic distance can be download from [here](http://www.uwyo.edu/dbmcd/molmark/gendisteqns.pdf)

Reynolds distance:
$$
\theta_w = \sqrt{\frac{\sum_l\sum_u (X_u -Y_u)^2}{2\sum_l(1-\sum_u X_u Y_u)}}
$$

R codes as below:
{% highlight r %}
reynolds_distance=function(Data) 
{
  popnames=rownames(Data)
  npop=nrow(Data) 
  nloc=ncol(Data) 
  dist=matrix(0,nrow=npop,ncol=npop) 
  for (i in 1:(npop-1)) 
  { 
    for (j in (i+1):npop) 
    { 
      pi=Data[i,] 
      pj=Data[j,] 
      
      dist[i,j]=sqrt(sum( (pi-pj)**2+(pj-pi)**2 )/(2*sum(1-(pi*pj+(1-pi)*(1-pj)))) )
    } 
  } 
  dist=dist+t(dist) 
  rownames(dist)=colnames(dist)=popnames
  return(dist) 
}

###example
set.seed(44)
pop1 <- runif(12)
pop2 <- runif(12)
df <- rbind(pop1,pop2)
dist <- reynolds_distance(df)
dist
#          pop1      pop2
# pop1 0.0000000 0.4416244
# pop2 0.4416244 0.0000000
{% endhighlight %}