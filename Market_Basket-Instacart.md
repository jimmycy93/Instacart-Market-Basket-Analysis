Loading packages
----------------

    suppressMessages(library(dplyr)) 
    suppressMessages(library(arules)) 
    suppressMessages(library(arulesViz)) 
    suppressMessages(library(stringr)) 

Reading and transforming data
-----------------------------

First we will read the data and take a brief look at it.

    setwd("C:/Users/Jimmy Chen/Desktop/Skyrec/FYI/Market Basket")
    orders<-read.csv("Market Basket-Instacart.csv")
    head(orders)

    ##   order_id product_id               product_name user_id
    ## 1     6695          1 Chocolate Sandwich Cookies    1540
    ## 2    48361          1 Chocolate Sandwich Cookies  194636
    ## 3    63770          1 Chocolate Sandwich Cookies     751
    ## 4    75339          1 Chocolate Sandwich Cookies  142585
    ## 5   240996          1 Chocolate Sandwich Cookies   57938
    ## 6   253400          1 Chocolate Sandwich Cookies   21054

Next we will read the data as a 'transaction' object in order for arules
to read it, as well as setting the item info as product name.

    trans<-read.transactions("Market Basket-Instacart.csv", format = "single", sep=",",cols = c("order_id","product_id"))
    orders<-orders[order(as.character(orders$product_id)),]
    items<-as.character(unique(orders$product_name)) #get the corresponding name for product id
    trans@itemInfo<-data.frame(labels=items)
    inspect(trans[1:2])

    ##     items                                           transactionID
    ## [1] {Organic Celery Hearts,                                      
    ##      Organic 4% Milk Fat Whole Milk Cottage Cheese,              
    ##      Bag of Organic Bananas,                                     
    ##      Organic Whole String Cheese,                                
    ##      Lightly Smoked Sardines in Olive Oil,                       
    ##      Organic Hass Avocado,                                       
    ##      Bulgarian Yogurt,                                           
    ##      Cucumber Kirby}                                       1     
    ## [2] {Corn Tortillas,                                             
    ##      I Heart Baby Kale,                                          
    ##      Organic Baby Spinach,                                       
    ##      Organic Yellow Onion,                                       
    ##      No Salt Added Black Beans,                                  
    ##      Total 2% All Natural Plain Greek Yogurt,                    
    ##      Original Hummus,                                            
    ##      Extra Virgin Olive Oil,                                     
    ##      Unscented Long Lasting Stick Deodorant,                     
    ##      Ground Cumin,                                               
    ##      Wheat Sandwich Thins,                                       
    ##      Gala Apples,                                                
    ##      Organic Baby Carrots,                                       
    ##      Garnet Sweet Potato (Yam),                                  
    ##      Snack Sticks Chicken & Rice Recipe Dog Treats}        100000

Item Frequency
--------------

Before extracting rules from transaction data, we want to first look at
the frequency of which item appeared to gain knowledge of data.

    itemFrequencyPlot(trans,topN=20,type="absolute")

![](Market_Basket-Instacart_files/figure-markdown_strict/unnamed-chunk-4-1.png)

    item_frequency<-itemFrequency(trans,type='absolute')
    summary(item_frequency)

    ##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
    ##    1.000    1.000    2.000    7.482    5.000 1973.000

Some quick thoughts from above: - Many of the most popular items are
fruits.

-   The frequency differs a lot between items, some items are much more
    popular than others.

-   75% of the items appear equal or less than 5 times, which is a very
    small ratio of 14000 transactions. in total, this must be considered
    when deciding minimum support.

Rules forming
-------------

An association rule, for example {diaper-&gt;beer} indicates that if a
person buys diaper, it is likely to occur that he will also buy beer.The
three most frequently used evaluation metric in association
rules{X-&gt;Y} are:

-   Support : Fraction of transactions that both X and Y appears, using
    the above example, it means the fraction of transactions that
    include both diaper and beer.

-   Confidence : Given that X is purchased, the conditional probability
    that Y will also be purchased, for example that 2 out of 5 people
    that purchased diaper also bought beer, the confidence is then 2/5
    = 0.4.

-   Lift : The effect X has on the occurence of Y, comparing to the
    expect occurence of Y, for example, a lift of 4 indicates that given
    X, Y is 4 times more likely to appear than expected.

In my opinion, lift is the most important metric, however, support and
confidence also needs to be considered when generating rules. A rule
with high lift but little support might be a coincidence or a flavor of
some particular customer, and that a rule with high lift but little
confidence means that the occurence of Y is still too unlikely comparing
to other items, therefore hard to create value in business. With these
in mind, my way of creating rules would be setting a minimum support and
confidence, then sorting by lift to find the most important rules among
all.

We will choose a minimum support of 0.001 and a minimum confidence of
0.4, and setting the maximum length of rules to be 5.

    rules <- apriori(trans, parameter = list(supp = 0.001, conf = 0.4,maxlen = 5))
    rules<-sort(rules, by="lift", decreasing=TRUE)
    options(digits = 3)

Inspecting the rules
--------------------

    inspect(rules[1:4])

    ##     lhs                                           rhs                                             support confidence  lift count
    ## [1] {Fat Free Blueberry Yogurt}                => {Fat Free Strawberry Yogurt}                    0.00114      0.516 157.3    16
    ## [2] {Total 0% Blueberry Acai Greek Yogurt}     => {Total 0% Raspberry Yogurt}                     0.00107      0.517 139.5    15
    ## [3] {Nonfat Icelandic Style Strawberry Yogurt} => {Icelandic Style Skyr Blueberry Non-fat Yogurt} 0.00135      0.442  80.5    19
    ## [4] {Yellow Bell Pepper,                                                                                                        
    ##      Banana}                                   => {Orange Bell Pepper}                            0.00128      0.500  32.6    18

    summary(rules)

    ## set of 191 rules
    ## 
    ## rule length distribution (lhs + rhs):sizes
    ##   2   3   4 
    ##  15 156  20 
    ## 
    ##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    ##    2.00    3.00    3.00    3.03    3.00    4.00 
    ## 
    ## summary of quality measures:
    ##     support          confidence         lift           count   
    ##  Min.   :0.00107   Min.   :0.400   Min.   :  2.8   Min.   :15  
    ##  1st Qu.:0.00114   1st Qu.:0.432   1st Qu.:  3.7   1st Qu.:16  
    ##  Median :0.00135   Median :0.469   Median :  4.3   Median :19  
    ##  Mean   :0.00157   Mean   :0.486   Mean   :  7.2   Mean   :22  
    ##  3rd Qu.:0.00178   3rd Qu.:0.520   3rd Qu.:  5.7   3rd Qu.:25  
    ##  Max.   :0.00570   Max.   :0.767   Max.   :157.3   Max.   :80  
    ## 
    ## mining info:
    ##   data ntransactions support confidence
    ##  trans         14023   0.001        0.4

    plot(rules,method = 'scatterplot', measure = c("support","confidence"), shading = "lift")

![](Market_Basket-Instacart_files/figure-markdown_strict/unnamed-chunk-7-1.png)
From the summary and scatter plot of rules:

-   Most of the rules are length of 3.
-   Most of the rules have support between 0.001 and 0.002.
-   There seems to be a few rules that are very strong.

<!-- -->

    plot(rules[1:20], method="graph",control = list(cex=0.5))

![](Market_Basket-Instacart_files/figure-markdown_strict/unnamed-chunk-8-1.png)
The graph above enable us to visualize the cluster of items. From the
graph, we can see some interesting rules, just to mention a few, along
with some new questions:

-   Trivial : A person who buy Whole Milk Yogurt tend to purchase a
    bottle of Whole Milk, this makes sense to me. Does the other way
    around work the same and what can we do with this combination?

-   Mysterious : Fat Free Blueberry yogurt and Fat Free Strawberry
    yogurt, as well as 0% Blueberry Acai Greek Yogurt and 0% Raspberry
    Yogurt have an unusually strong lift and little support which seems
    strange to me. Does the relationship really exist or only because of
    some particular customers?

-   Useful : There is a group of items that is associated with limes,
    there seems to exist some unknown but useful relationship. Are they
    all correlated and does this phenomenon have any business potential?

Answer the questions
--------------------

#### 1. Confirm the trivial

    yogurt_milk<-apriori(data=trans, parameter=list(supp=0.001,conf = 0.4), 
                   appearance = list(rhs="Organic Whole Milk",default="lhs"),
                   control = list(verbose=F))
    inspect(yogurt_milk)

    ##     lhs                          rhs                  support confidence
    ## [1] {Whole Milk Plain Yogurt} => {Organic Whole Milk} 0.00171 0.414     
    ##     lift count
    ## [1] 10.5 24

    support_list<-rules@quality$support
    support_rank<-length(support_list[support_list>yogurt_milk@quality$support])
    paste("The rule has the",support_rank,"largest support among 191 rules extracted")

    ## [1] "The rule has the 49 largest support among 191 rules extracted"

    lift_list<-rules@quality$lift
    lift_rank<-length(lift_list[lift_list>yogurt_milk@quality$lift])
    paste("The rule has the",lift_rank,"largest lift among 191 rules extracted")

    ## [1] "The rule has the 14 largest lift among 191 rules extracted"

    confidence_list<-rules@quality$confidence
    confidence_rank<-length(confidence_list[confidence_list>yogurt_milk@quality$confidence])
    paste("The rule has the",confidence_rank,"largest confidence among 191 rules extracted")

    ## [1] "The rule has the 169 largest confidence among 191 rules extracted"

This rule is a strong rule among all rules, with a nice support and a
large lift over 10, however it has a slightly low confidence. Let's see
if the other way around, {Whole Milk Plain Yogurt} =&gt; {Organic Whole
Milk} works the same.

    milk_yogurt<-apriori(data=trans, parameter=list(supp=0.001,conf = 0.01), 
                   appearance = list(rhs="Whole Milk Plain Yogurt",default="lhs"),
                   control = list(verbose=F))
    inspect(milk_yogurt)

    ##     lhs                     rhs                       support confidence
    ## [1] {Organic Whole Milk} => {Whole Milk Plain Yogurt} 0.00171 0.0434    
    ##     lift count
    ## [1] 10.5 24

Support and Lift are the same because they are symmetric. However, the
confidence for the rule is only 0.04, indicating that buying Organic
Whole Milk would not usually lead to buying Whole Milk Plain Yogurt.
Therefore, a business strategy might be promoting Whole Milk Plain
Yogurt with a little box of Oragnic Whole Milk as complementary.

#### 2. Test the mysterious

    fat_free_yogurt<-apriori(data=trans, parameter=list(supp=0.001,conf = 0.4), 
                   appearance = list(rhs="Fat Free Strawberry Yogurt",default="lhs"),
                   control = list(verbose=F))
    inspect(fat_free_yogurt)

    ##     lhs                            rhs                          support confidence lift count
    ## [1] {Fat Free Blueberry Yogurt} => {Fat Free Strawberry Yogurt} 0.00114      0.516  157    16

    find_user_id<-function(id){
      cond<-all(c("Fat Free Strawberry Yogurt","Fat Free Blueberry Yogurt")%in%orders[orders$order_id==id,"product_name"])
      return(ifelse(cond,unique(orders[orders$order_id==id,"user_id"]),-1))
    }
    ids<-sapply(unique(orders$order_id),find_user_id)
    ids<-ids[ids>=0]
    paste("There are",length(unique(ids)),"unique users in",length(ids),"total users that purchased the combination")

    ## [1] "There are 16 unique users in 16 total users that purchased the combination"

It seems that there are all customers that bought the Fat Free
combination are unique in the data time range. However, there might be
other reasons that caused the high lift, such as an existing promotion
or so, which needs further investigation.

#### 3. Investigate the useful

    lime<-apriori(data=trans, parameter=list(supp=0.001,conf = 0.4), 
                   appearance = list(rhs="Limes",default="lhs"),
                   control = list(verbose=F))
    inspect(lime)

    ##      lhs                                        rhs     support confidence
    ## [1]  {Jalapeno Peppers,Organic Cilantro}     => {Limes} 0.00121 0.500     
    ## [2]  {Jalapeno Peppers,Organic Avocado}      => {Limes} 0.00107 0.517     
    ## [3]  {Jalapeno Peppers,Large Lemon}          => {Limes} 0.00157 0.564     
    ## [4]  {Jalapeno Peppers,Organic Baby Spinach} => {Limes} 0.00128 0.600     
    ## [5]  {Bunched Cilantro,Large Lemon}          => {Limes} 0.00143 0.500     
    ## [6]  {Organic Baby Spinach,Bunched Cilantro} => {Limes} 0.00107 0.517     
    ## [7]  {Organic Cilantro,Asparagus}            => {Limes} 0.00107 0.556     
    ## [8]  {Organic Cilantro,Organic Avocado}      => {Limes} 0.00178 0.410     
    ## [9]  {Organic Cilantro,Large Lemon}          => {Limes} 0.00164 0.434     
    ## [10] {Organic Baby Spinach,Organic Cilantro} => {Limes} 0.00185 0.406     
    ## [11] {Organic Garlic,Organic Avocado}        => {Limes} 0.00150 0.404     
    ##      lift  count
    ## [1]  11.22 17   
    ## [2]  11.61 15   
    ## [3]  12.66 22   
    ## [4]  13.46 18   
    ## [5]  11.22 20   
    ## [6]  11.61 15   
    ## [7]  12.46 15   
    ## [8]   9.20 25   
    ## [9]   9.74 23   
    ## [10]  9.11 26   
    ## [11]  9.06 21

From our rules, we can see a bunch of items that customers often
purchase with Limes, they are Jalapeno Peppers, Cilantro, Avocado, Baby
Spinach, Garlic, Asparagus, and Large Lemon. Feeding these names to
google gave me some special dishes such as baby spinach avocado salad,
Garlic Asparagus with Lime, or Cilantro Lime Asparagus and Rice as
results. These are useful information that I don't know since I don't
cook at all, and it seems that the rules do exist, and that Limes or
Large Lemon are important ingredients for all these dishes. Therefore, a
business strategy is to place these items close together, while putting
Limes and Large Lemon far away from them, so that customers would have
to walk through other department to get them, thus increasing the chance
of other purchases.

Conclusion
----------

In this small project I only chose serveral rules to look into because
of time, but there is a lot more rules to work with. This is a truely
amazing dataset, thanks to Instacart, and there are many other things
that can be done to the dataset, such as determining different purchase
behavior of different time, or to build a simple recommendation system
base on the rules and collaborative filtering between users. Maybe next
time.
