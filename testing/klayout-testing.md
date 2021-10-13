Other Klayout Functions
================
Scott Alder
8/24/2021

``` r
library(reticulate)
library(dplyr)
library(readr)
library(tidyr)
library(ggplot2)
library(rgdal)
library(broom)
library(magick)
```

# Create dummy data

``` python
#### Python Code Chunk ####
import pya # "pya" is the actual module name, not "klayout"

# Create dummy layout data
layout = pya.Layout()
top = layout.create_cell("TOP")
l1 = layout.layer(1, 0)

# Repetition
x_reps = [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15]
y_reps = [1,2,3,4,5,6,7,8,9,10]
dims = [1000, 2000]
sep  = [100, 200]

for xr in x_reps:
  for yr in y_reps:
    top.shapes(l1).insert(
      pya.Box(
        0.0     + (xr - 1)*(sep[0] + dims[0]), # Left
        0.0     + (yr - 1)*(sep[1] + dims[1]), # Bottom
        dims[0] + (xr - 1)*(sep[0] + dims[0]), # Right
        dims[1] + (yr - 1)*(sep[1] + dims[1])  # Top
      )
    ) 
    
# Save as *.oas file
```

``` python
layout.write("test2.oas")
```

``` python
layout.write("test2.dxf")
```

``` r
#### R Code Chunk ####
layout_data <- 
  readOGR("test2.dxf") %>% 
  tidy() %>% 
  rename(x=long, y=lat)
```

    ## OGR data source with driver: DXF 
    ## Source: "C:\Users\gscot\Desktop\klayout-api\klayout-api-testing\testing\test2.dxf", layer: "entities"
    ## with 150 features
    ## It has 6 fields

``` r
theme_set(theme_bw())
g <- 
  layout_data %>% 
  ggplot(aes(x, y, group=group)) +
  geom_polygon(fill="black", color=NA) +
  coord_fixed() +
  scale_x_continuous(n.breaks = 10) +
  scale_y_continuous(n.breaks = 10)

print(g)
```

![](klayout-testing_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

# Clipping

``` python
import pya
layout = pya.Layout.new()
layout.read("test2.oas")

top_cell = layout.cell("TOP")

bb_array = [
  pya.Box.new(2000, 2500, 7000, 7500),
  pya.Box.new(12000, 12500, 15000, 15500)
]

clipped_layout = pya.Layout.new()
clipped_layout.dbu = layout.dbu
for layer_id in layout.layer_indices():
  clipped_layout.insert_layer_at(layer_id, layout.get_info(layer_id))
  
clip_cells = layout.multi_clip_into(top_cell.cell_index(), clipped_layout, bb_array)  

clip_top_cell = clipped_layout.create_cell("TOP")  
for cc in clip_cells: 
  clip_top_cell.insert(pya.CellInstArray(cc, pya.Trans()))

clipped_layout.write("clipped_test2.oas")
clipped_layout.write("clipped_test2.dxf")
```

    ## <klayout.dbcore.LayerMap object at 0x0000000032434A98>
    ## <klayout.dbcore.Layout object at 0x0000000032434C78>
    ## <klayout.dbcore.Instance object at 0x0000000032434D68>
    ## <klayout.dbcore.Instance object at 0x0000000032434DE0>
    ## <klayout.dbcore.Layout object at 0x0000000032434C78>
    ## <klayout.dbcore.Layout object at 0x0000000032434C78>

``` r
theme_set(theme_bw())
g <- 
  readOGR("clipped_test2.dxf") %>% 
  tidy() %>% 
  rename(x=long, y=lat) %>% 
  ggplot(aes(x, y, group=group)) +
  geom_polygon(fill="black", color=NA) +
  coord_fixed() +
  scale_x_continuous(n.breaks = 10) +
  scale_y_continuous(n.breaks = 10)

print(g)
```

![](klayout-testing_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

    ## OGR data source with driver: DXF 
    ## Source: "C:\Users\gscot\Desktop\klayout-api\klayout-api-testing\testing\clipped_test2.dxf", layer: "entities"
    ## with 2 features
    ## It has 6 fields

# Image capture

``` r
knitr::knit_engines$set(lym = function(options) {
  # the source code is in options$code; just do
  # whatever you want with it
  lym_code <- paste(options$code, collapse = "\n") #glue::glue(options$code)
  ly_file <- options$layout_file
  
  tmp_py <- tempfile(fileext=".py")
  writeLines(lym_code, tmp_py)
  
  klayout_exe_path <-
    Sys.getenv("USERPROFILE") %>%
    paste0("\\AppData\\Roaming\\") %>%
    list.files("klayout_app.exe", recursive = T, full.names = T) %>%
    normalizePath()
  
  print(klayout_exe_path)
  print(ly_file)
  print(lym_code)
  
  # `<path to klayout_app.exe> <path to layout file> -r <path to Python script>`
  lym_output <- 
    paste(klayout_exe_path, ly_file, "-r", tmp_py) %>% # -r prevents GUI from opening
    system(intern=TRUE)
})
```

Saving image for position 4.5,9.5 to img1.pngSaving image for position
14.5,17.5 to img2.png

``` r
image_read("img1.png") %>% 
  plot()
```

![](klayout-testing_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

``` r
image_read("img2.png") %>% 
  plot()
```

![](klayout-testing_files/figure-gfm/unnamed-chunk-9-2.png)<!-- -->

# Coordinate transformations

``` python
import pya
layout = pya.Layout.new()
layout.read("test2.oas")

# The sequence of operations is: magnification, mirroring at x axis,
#   rotation, application of displacement.
# @param mag The magnification
# @param rot The rotation angle in units of degree
# @param mirrx True, if mirrored at x axis
# @param x The x displacement
# @param y The y displacement
t = pya.CplxTrans.new(20.0, 90, True, -30.0, 20.0)

layout.transform(t.to_trans())

layout.write("trans_test2.dxf")
```

    ## <klayout.dbcore.LayerMap object at 0x0000000032434B10>
    ## <klayout.dbcore.Layout object at 0x000000003242F048>
    ## <klayout.dbcore.Layout object at 0x000000003242F048>

``` r
theme_set(theme_bw())
g <- 
  readOGR("trans_test2.dxf") %>% 
  tidy() %>% 
  rename(x=long, y=lat) %>% 
  ggplot(aes(x, y, group=group)) +
  geom_polygon(fill="black", color=NA) +
  coord_fixed() +
  scale_x_continuous(n.breaks = 10) +
  scale_y_continuous(n.breaks = 10)

print(g)
```

![](klayout-testing_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

    ## OGR data source with driver: DXF 
    ## Source: "C:\Users\gscot\Desktop\klayout-api\klayout-api-testing\testing\trans_test2.dxf", layer: "entities"
    ## with 150 features
    ## It has 6 fields

# Combine filesdw

``` python
#### Python Code Chunk ####
import pya # "pya" is the actual module name, not "klayout"

# Create dummy layout data
layout = pya.Layout()
top = layout.create_cell("TOP")
l1 = layout.layer(2, 0)

# Repetition
origin = [0, 1000]
x_reps = [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15]
y_reps = [1,2,3,4,5,6,7,8,9,10]
dims = [1000, 200]
sep  = [100, 2000]

for xr in x_reps:
  for yr in y_reps:
    top.shapes(l1).insert(
      pya.Box(
        origin[0] + 0.0     + (xr - 1)*(sep[0] + dims[0]), # Left
        origin[1] + 0.0     + (yr - 1)*(sep[1] + dims[1]), # Bottom
        origin[0] + dims[0] + (xr - 1)*(sep[0] + dims[0]), # Right
        origin[1] + dims[1] + (yr - 1)*(sep[1] + dims[1])  # Top
      )
    ) 
    
# Save as *.oas file
```

``` python
layout.write("test3.oas")
```

``` python
layout.write("test3.dxf")
```

``` r
theme_set(theme_bw())
g <- 
  ggplot(data.frame(), aes(x, y, group=group)) +
  # geom_polygon(
  #   data = 
  #     readOGR("test2.dxf") %>% 
  #       tidy() %>% 
  #       rename(x=long, y=lat),
  #   fill="red", color=NA
  # ) +
  geom_polygon(
    data = 
      readOGR("test3.dxf") %>% 
      tidy() %>% 
      rename(x=long, y=lat),
    fill="black", color=NA
  ) +
  coord_fixed() +
  scale_x_continuous(n.breaks = 10) +
  scale_y_continuous(n.breaks = 10)

print(g)
```

![](klayout-testing_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

    ## OGR data source with driver: DXF 
    ## Source: "C:\Users\gscot\Desktop\klayout-api\klayout-api-testing\testing\test3.dxf", layer: "entities"
    ## with 150 features
    ## It has 6 fields

    #### Python Code Chunk ####
    import pya # "pya" is the actual module name, not "klayout"

    # Create dummy layout data
    layout_2 = pya.Layout()
    layout_2.read("test2.oas")
    layout_3 = pya.Layout()
    layout_3.read("test3.oas")

    # layout_4 = pya.Layout()
    # layout_4.insert_layer_at(layout_2.layer_indices())
    # layout_4.insert_layer_at(layout_3.layer_indices())

    merged_layout = pya.Layout.new()
    merged_layout.dbu = layout_2.dbu # assume 2 and 3 have equal DBU

    # Layers
    for layer_id in layout_2.layer_indices():
      merged_layout.insert_layer_at(layer_id, layout_2.get_info(layer_id))

    for layer_id in layout_3.layer_indices():
      merged_layout.insert_layer_at(layer_id, layout_3.get_info(layer_id))
      
    # Cells
    # clip_cells = layout.multi_clip_into(top_cell.cell_index(), clipped_layout, bb_array)  
    cells_2 = layout_2.each_cell()
    cells_3 = layout_3.each_cell()

    merged_top_cell = clipped_layout.create_cell("TOP")
    for cc in clip_cells:
      merged_top_cell.insert(pya.CellInstArray(cc, pya.Trans()))
      

    # top_cell = layout.cell("TOP")
    # 
    # bb_array = [
    #   pya.Box.new(2000, 2500, 7000, 7500),
    #   pya.Box.new(12000, 12500, 15000, 15500)
    # ]
    # 
    # clipped_layout = pya.Layout.new()
    # clipped_layout.dbu = layout.dbu
    # for layer_id in layout.layer_indices():
    #   clipped_layout.insert_layer_at(layer_id, layout.get_info(layer_id))
    #   
    # clip_cells = layout.multi_clip_into(top_cell.cell_index(), clipped_layout, bb_array)  
    # 
    # clip_top_cell = clipped_layout.create_cell("TOP")  
    # for cc in clip_cells: 
    #   clip_top_cell.insert(pya.CellInstArray(cc, pya.Trans()))
    # 
    # clipped_layout.write("clipped_test2.oas")
    # clipped_layout.write("clipped_test2.dxf")
