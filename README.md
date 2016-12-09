# pax



### Premable

... pax: population analysis in unix. This is a resuscitation of an old name - but likely going to do more.

Have you wished for a painless way to generate survey indices. Give the following a try.

### Needed packages


```r
devtools::install_github("fishvice/husky", dependencies = FALSE) # A temporary measure
devtools::install_github("fishvice/pax", dependencies = FALSE)
```

In addition it is assumed you have the __fjolst__-package.

### Loading needed libraries


```r
library(tidyverse)
library(fjolst)
library(pax)
```

### An example

#### Specify the species


```r
SPECIES <- 1
```

#### SMB all stations

OK, here you have to have some bacground knowledge. But we select "all" SMB stations:


```r
Station <- 
  husky::STODVAR %>% 
  bind_rows() %>% 
  filter(tognumer %in% 1:39) %>% 
  select(id = synis.id, year = ar, towlength = toglengd, strata = newstrata) %>% 
  mutate(towlength = pax:::trim_towlength(towlength),
         mult = 1) %>% 
  arrange(id)
```

And get the appropriate strata as well (the old strata, which is not the same as the very old strata):


```r
Stratas <-
  husky::stratas_df %>% 
  select(strata, area = rall.area)
```

And then we leave the rest to the `smx:calc_length_indices`-function:


```r
res <- calc_length_indices(Station, Stratas, SPECIES)
```

#### Visualization

Extracting the tidy information and preparing the plot:

```r
tidy_fixed <-
  res$aggr %>% 
  filter(length == 5) %>% 
  mutate(source = "tidy")
p <- 
  tidy_fixed %>% 
  select(year, cb, cb.cv) %>% 
  ggplot(aes(year, cb)) +
  geom_pointrange(aes(ymin = cb * (1 - cb.cv),
                      ymax = cb * (1 + cb.cv)),
                  colour = "red", lwd = 1) +
  geom_line(colour = "red", lwd = 1) +
  scale_colour_brewer(palette = "Set1") +
  labs(x = NULL, y = NULL,
       title = "Haddock: Spring survey biomass index",
       subtitle = "All stations")
```

Lets add the official calculations for comparison:

```r
attach("/net/hafkaldi/export/u2/reikn/Splus5/SMB/Allindices.RData")
mri <- 
  All.indices %>% 
  filter(species == SPECIES,
         svaedi == "Heild",
         lengd == 5,
         type == "all") %>% 
  select(year = ar, cb = bio.staerri, cb.cv = cv.bio.staerri, type) %>% 
  mutate(source = "mri")

p +
  geom_pointrange(data = mri, 
                  aes(ymin = cb * (1 - cb.cv),
                      ymax = cb * (1 + cb.cv)),
                  colour = "yellow") +
  geom_line(data = mri, colour = "yellow") +
  labs(x = NULL, y = NULL,
       title = "Biomass indices",
       subtitle = "Comparison of the tidyverse (red) and the Bible (grey)")
```

![](README_files/figure-html/plot1-1.png)<!-- -->

#### Something to worry about?


```r
tidy_fixed %>% 
  select(year, cb) %>% 
  left_join(mri %>% select(year, cb2 = cb)) %>% 
  mutate(diff = (cb - cb2)/cb2 * 100) %>% 
  summary()
```

```
##       year            cb              cb2              diff       
##  Min.   :1985   Min.   :161148   Min.   :160877   Min.   :0.1020  
##  1st Qu.:1993   1st Qu.:242193   1st Qu.:241671   1st Qu.:0.1872  
##  Median :2000   Median :289394   Median :288828   Median :0.2158  
##  Mean   :2000   Mean   :327553   Mean   :326905   Mean   :0.1977  
##  3rd Qu.:2008   3rd Qu.:388204   3rd Qu.:387380   3rd Qu.:0.2161  
##  Max.   :2016   Max.   :627611   Max.   :626306   Max.   :0.2161
```

So the tidyverse is 0.2% higher than the mri - some may want to dig into that.

### Another example, this time only for one sex

Here lets try to calculate the index for Grásleppa. We use the same station set and strata as above. Hence only need to do:

```r
res <- calc_length_indices(Station, Stratas, 48, Sex = 2)
# and a quick plot
res$aggr %>% 
  filter(length == 5) %>% 
  select(year, cb, cb.cv) %>% 
  ggplot(aes(year, cb)) +
  geom_pointrange(aes(ymin = cb * (1 - cb.cv),
                      ymax = cb * (1 + cb.cv)),
                  colour = "red", lwd = 1) +
  geom_line(colour = "red", lwd = 1) +
  scale_colour_brewer(palette = "Set1") +
  labs(x = NULL, y = NULL,
       title = "Grásleppa: Spring survey biomass index",
       subtitle = "All stations")
```

![](README_files/figure-html/plot2-1.png)<!-- -->

### One more, this time for a "non-standard" species


```r
spe88 <- calc_length_indices(Station, Stratas, 88, lwcoeff = c(0.01, 3))
spe562 <- calc_length_indices(Station, Stratas, 562, lwcoeff = c(0.01, 3))
spe150 <- calc_length_indices(Station, Stratas, 150, lwcoeff = c(0.01, 3))
spe71 <- calc_length_indices(Station, Stratas, 71, lwcoeff = c(0.01, 3))
spe88$aggr %>% mutate(Species = "Rauða sævesla") %>% 
  bind_rows(spe562$aggr %>% mutate(Species = "Ljóskjafta")) %>% 
  bind_rows(spe150$aggr %>% mutate(Species = "Svartgóma")) %>% 
  bind_rows(spe71$aggr %>% mutate(Species = "Ískóð")) %>% 
  filter(length == 140) %>% 
  select(year, Species, cn, cn.cv) %>% 
  ggplot(aes(year, cn)) +
  geom_pointrange(aes(ymin = cn * (1 - cn.cv),
                      ymax = cn * (1 + cn.cv)),
                  colour = "red", lwd = 1) +
  geom_line(colour = "red", lwd = 1) +
  scale_colour_brewer(palette = "Set1") +
  facet_wrap(~ Species, scale = "free_y") +
  labs(x = NULL, y = NULL,
       title = "Spring survey abundance index",
       subtitle = "All stations")
```

```
## Warning: Removed 58 rows containing missing values (geom_pointrange).
```

![](README_files/figure-html/plot3-1.png)<!-- -->

So we have and invasion of Ljóskjafta and Svartgóma in recent years. That must explain the mackerel invasion :-)

NOTE: There is a bug when it comes to the biomass estimates of the two last species. And one needs to double test for when some species were only counted.
