Extinctions Unit
================
Riya Shrestha, Kristin Liu

## Extinctions Module

*Are we experiencing the sixth great extinction?*

What is the current pace of extinction? Is it accelerating? How does it
compare to background extinction rates?

The current pace of extinction is vastly accelerating. Even under
extremely conservative assumptions, with a background rate of 2 mammal
extinctions per 10,000 species per 100 years (2 E/MSY) which is two
times higher than previous estimates, we can see that the mean rate of
vertabrate loss that has been seen in the past century is up to 100
times higher than the background rate. Under the assumptions of the 2
E/MSY rate, the amount that disappeared in the last 100 years would have
taken 800 to 10,000 years to become extinct. This is showing how the
pace of extinction is accelerating over time. It also serves to show how
we are rapidly lowering th biodiversity on the planet and are hurtling
toward the 6th great extinction.

## Background

  - [Section Intro Video](https://youtu.be/QsH6ytm89GI)
  - [Ceballos et al (2015)](http://doi.org/10.1126/sciadv.1400253)

Our focal task will be to reproduce the result from Ceballos and
colleagues showing the recent increase in extinction rates relative to
the background rate:

![](https://espm-157.carlboettiger.info/img/extinctions.jpg)

## Additional references:

  - <http://www.hhmi.org/biointeractive/biodiversity-age-humans> (Video)
  - [Barnosky et al. (2011)](http://doi.org/10.1038/nature09678)
  - [Pimm et al (2014)](http://doi.org/10.1126/science.1246752)
  - [Sandom et al (2014)](http://dx.doi.org/10.1098/rspb.2013.3254)

## Code for extracting species and extinct data

#### Species List

``` r
base <- "https://apiv3.iucnredlist.org/api/v3/species/"
page <- "page/"
page_number <- 0:14
query <- "?token="
token <- "9bb4facb6d23f48efbf424bb05c0c1ef1cf6f468393bc745d42179ac4aca5fee"
  
urls <- paste0(base, page, page_number, query, token)

if(!file.exists("species_request.rds")){
  species_request <- map(urls, GET)
  saveRDS(species_request, "species_request.rds")
}

species_request <- readRDS("species_request.rds")
```

``` r
contents <- map(species_request, content, as = "text")
species <- map(contents, fromJSON)
species_df <- map_dfr(species, "result")
```

#### Finding total number of species in each class from original data.

``` r
total_class_count <- species_df %>% 
  count(class_name) %>%
  rename(total_count = n)
head(total_class_count)
```

``` 
        class_name total_count
1   ACTINOPTERYGII       21194
2   AGARICOMYCETES         425
3         AMPHIBIA        7216
4    ANDREAEOPSIDA           3
5 ANTHOCEROTOPSIDA           2
6         ANTHOZOA         868
```

#### Filtering by extinct species and then class, finding percentage of extinction

``` r
extinct <- filter(species_df, category == "EX")

extinct_count <- extinct %>% 
  count(class_name) %>%
  rename(extinct_species = n) %>%
  left_join(total_class_count, by = "class_name")

percentage_extinction <- extinct_count %>%
  mutate(percentage_extinct = extinct_species / total_count)
head(percentage_extinction)
```

``` 
      class_name extinct_species total_count percentage_extinct
1 ACTINOPTERYGII              86       21194        0.004057752
2       AMPHIBIA              35        7216        0.004850333
3      ARACHNIDA               9         393        0.022900763
4           AVES             159       11158        0.014249866
5       BIVALVIA              35         834        0.041966427
6      BRYOPSIDA               4         171        0.023391813
```

#### Finding year of extinction for each species.

``` r
base <- "https://apiv3.iucnredlist.org/api/v3/species/narrative/"
sci_name <- extinct$scientific_name
token <- "?token=9bb4facb6d23f48efbf424bb05c0c1ef1cf6f468393bc745d42179ac4aca5fee"
url <- paste0(base, sci_name, token)

if(!file.exists("year_extinct_request.rds")){
  year_extinct_request <- map(url, GET)
  saveRDS(year_extinct_request, "year_extinct_request.rds")
}

year_extinct_request <- readRDS("year_extinct_request.rds")
```

``` r
text_year_extinct <- map(year_extinct_request, content, as = "text")
json <- map_dfr(text_year_extinct, fromJSON)
txt <- json$result$rational

extract_last_year <- function(txt){
  all_years <- str_extract_all(txt, "\\d{4}")
  map_dbl(all_years, ~ max(as.integer(.)))
}

head(extract_last_year(txt))
```

    [1] 1975   NA   NA   NA   NA   NA

``` r
last_year_data <- json$result %>% 
  mutate(last_year = extract_last_year(rationale)) %>%
  left_join(extinct, by = c("species_id" = "taxonid")) %>%
  select(species_id, last_year, kingdom_name, phylum_name, class_name, order_name, family_name, genus_name, scientific_name,
         category, main_common_name)
head(last_year_data)
```

``` 
  species_id last_year kingdom_name phylum_name     class_name      order_name
1         73      1975     ANIMALIA    CHORDATA ACTINOPTERYGII   CYPRINIFORMES
2         82        NA     ANIMALIA  ARTHROPODA        INSECTA   EPHEMEROPTERA
3        167        NA     ANIMALIA    MOLLUSCA     GASTROPODA STYLOMMATOPHORA
4        170        NA     ANIMALIA    MOLLUSCA     GASTROPODA STYLOMMATOPHORA
5        173        NA     ANIMALIA    MOLLUSCA     GASTROPODA STYLOMMATOPHORA
6        174        NA     ANIMALIA    MOLLUSCA     GASTROPODA STYLOMMATOPHORA
          family_name      genus_name            scientific_name category
1          CYPRINIDAE        Mirogrex          Mirogrex hulensis       EX
2 ACANTHAMETROPODIDAE Acanthametropus Acanthametropus pecatonica       EX
3      ACHATINELLIDAE     Achatinella     Achatinella abbreviata       EX
4      ACHATINELLIDAE     Achatinella         Achatinella buddii       EX
5      ACHATINELLIDAE     Achatinella         Achatinella caesia       EX
6      ACHATINELLIDAE     Achatinella          Achatinella casta       EX
         main_common_name
1              Hula Bream
2 Pecatonica River Mayfly
3                    <NA>
4                    <NA>
5                    <NA>
6                    <NA>
```

## Arranging by class and calculating percentage of extinction

``` r
mammals <- last_year_data %>%
  filter(class_name == "MAMMALIA") %>%
  arrange(last_year) %>%
  left_join(percentage_extinction, by = "class_name") %>%
  mutate(number = row_number()) %>%
  mutate(cumulative_percentage_extinct =  number / total_count)
birds <- last_year_data %>%
  filter(class_name == "AVES") %>%
  arrange(last_year) %>%
  left_join(percentage_extinction, by = "class_name") %>%
  mutate(number = row_number()) %>%
  mutate(cumulative_percentage_extinct =  number / total_count)
amphibians <- last_year_data %>%
  filter(class_name == "AMPHIBIA") %>%
  arrange(last_year) %>%
  left_join(percentage_extinction, by = "class_name") %>%
  mutate(number = row_number()) %>%
  mutate(cumulative_percentage_extinct =  number / total_count)
reptiles <- last_year_data %>%
  filter(class_name == "REPTILIA") %>%
  arrange(last_year) %>%
  left_join(percentage_extinction, by = "class_name") %>%
  mutate(number = row_number()) %>%
  mutate(cumulative_percentage_extinct =  number / total_count)
vertebrates_extinct <- last_year_data %>% 
  filter(class_name %in% c("MAMMALIA", "ACTINOPTERYGII", "AMPHIBIA", "REPTILIA", 
                           "AVES")) %>%
  arrange(last_year) %>%
  left_join(percentage_extinction, by = "class_name") %>%
  mutate(number = row_number()) %>%
  mutate(total_vertebrates = 55166) %>%
  mutate(cumulative_percentage_extinct =  number / total_vertebrates)
```

## Graphing extinction plot

``` r
colors <- c("mammals" = "blue", "birds" = "red", "amphibians" = "green", "reptiles" = "purple", "vertebrates" = "orange",
            "background rate" = "black")

extinction_graph <- ggplot() +
  geom_line(data = mammals, aes(x = last_year, y = cumulative_percentage_extinct*100, color = "mammals")) +
  geom_line(data = birds, aes(x = last_year, y = cumulative_percentage_extinct*100, color = "birds")) + 
  geom_line(data = amphibians, aes(x = last_year, y = cumulative_percentage_extinct*100, color = "amphibians")) +
  geom_line(data = reptiles, aes(x = last_year, y = cumulative_percentage_extinct*100, color = "reptiles")) +
  geom_line(data = vertebrates_extinct, aes(x = last_year, y = cumulative_percentage_extinct*100, color = "vertebrates")) +
  geom_abline(slope = 0.0002, intercept = -0.3, linetype = "dashed", aes(color = "background rate")) + 
  labs(title = "Cumulative vertebrate species recorded as extinct by the IUCN", x = "Year of extinction", 
       y = "Cumulative exctinctions as % of IUCN-evaluated species") +
  labs(colour = "Class") +
  scale_color_manual(values = colors)
extinction_graph
```

![](extinction-assignment_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

## Analysis of data and graph

#### Explanation of our graph and data versus graph and qualitative conclusions of Cerbellos

As you can see through this graph we attempted to recreate the graph
from Cerbellos et al. titled “Accelerated modern human–induced species
losses: Entering the sixth mass extinction.” In the graph we made, there
are some distinct changes from the initial graph. Instead of the
categories of vertebrates, mammals, other vertebrates, and birds, we
used the categories of mammals, birds, amphibians, reptiles, and
vertebrates. In terms of organization, we felt it was much more readable
and relevant to sort by classes in the vertebrate category and then an
overall vertebrate line. We also did not bin our data because we only
use four digit numeric numbers for the dates for the x-axis. If our team
had additional time to work more in depth on the project we would have
spent time using regular expressions to extract other dates front the
rationale that were not 4 digit numerics, like “18th century”, we would
have binned the data for our graph. The binning is also why our lines
look so different as we mapped out the lines by year rather than decade.

#### Overal analysis of extinction graph

Overall our graph shows very similar results to Cerbellos et al. We can
see in our graph that the rate of extinction among all the different
classes of animals is higher than our background rate of 2 mammal
extinctions per 10,000 species per 100 years as well as an increase in
the rate of extinction over time. All this points to a 6th mass
extinction in the works. As Barnosky et al. (2011) states, all of the
previous big 5 extinctions have had key synergies that may involve
unusual climate dynamics, atmospheric composition and abnormally
high-intensity ecological stressors that negatively affect many
different lineages. These are changes we are seeing today in our world
through the impacts of climate change and pollution as atmospheric
conditions change, CO2 levels rise, and the human population expands. As
Pimm et al (2014) states, the overarching driver of species extinction
is human population growth and increasing per capita consumption. The
only way to save the future of our plant as we know it is for us to find
solutions to the overpopulation and overconsumption problem humans on
earth are facing.
