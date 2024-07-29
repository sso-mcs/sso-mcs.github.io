---
layout: default
---

<!--Text can be **bold**, _italic_, or ~~strikethrough~~.

[Link to another page](./another-page.html).

There should be whitespace between paragraphs.

There should be whitespace between paragraphs. We recommend including a README, or a file with information about your project.-->

# Airbnbs in New York City
<div class="row">
    <div id="select" class="narrow-column">Filter by Borough:</div>
    <div id="listing" class="narrow-column"></div>
</div>
<div class="row">
    <div id="div_template" class="column"></div>
    <div id="div_list" class="column"></div>
</div>
<script>
async function init() {

    width = 400,
    height = 400;
    margin = 100;
    ind = 0;
    curr_ind = 0;

    // listings.csv of New York City at Inside Airbnb   
    file = "https://raw.githubusercontent.com/sso-mcs/sso-mcs.github.io/main/data/listings.csv"


    /*    drop down list    */
    var select = d3.select("#select")
          .append("select")
          
    var data = await d3.csv(file);
    var neighbourhood_group = d3.nest() 
          .key(function (d) { return d.neighbourhood_group; }) 
          .sortKeys(d3.ascending) 
          .entries(data);
          
    var ng_data = [];
    for (var i=0; i < neighbourhood_group.length; i++) {
        var home = 0, private = 0, shared = 0, hotel = 0;
        for (var j=0; j < neighbourhood_group[i].values.length; j++) {
            if(neighbourhood_group[i].values[j].room_type=="Entire home/apt"){
                home++;
            }else if(neighbourhood_group[i].values[j].room_type=="Private room"){
                private+=1;
            }else if(neighbourhood_group[i].values[j].room_type=="Shared room"){
                shared+=1;
            }else if(neighbourhood_group[i].values[j].room_type=="Hotel room"){
                hotel+=1;
            }  
        }
        ng_data[i] = [{"room_type": "Entire home/apt", "count": home},
                      {"room_type": "Private room", "count": private},
                      {"room_type": "Shared room", "count": shared},
                      {"room_type": "Hotel room", "count": hotel}];
    }
            
    total_listings = d3.count(data, d => d.id);
    selected_data = neighbourhood_group[0];
    selected_listings = selected_data.values.length;
    draw_annotations(total_listings, selected_listings);
    
    select
      .on("change", function(d) {
        curr_ind = d3.select(this).property("value");
        selected_data = neighbourhood_group[curr_ind];
        selected_listings = selected_data.values.length;
        draw_annotations(total_listings, selected_listings);
        update_chart(ng_data[curr_ind]);
        d3.selectAll("#div_list > *").remove();
      });
    
    select.selectAll("option")
      .data(neighbourhood_group)
      .enter()
        .append("option")
        .attr("value", d => ind++)
        .text(d => d.key);
    /*    drop down list    */
    
    
    // canvas scene
    var svg = d3.select("#div_template")
        .append("svg")
        .attr("width", width + margin + margin)
        .attr("height", height + margin + margin)
        .append("g")
        .attr("transform","translate(0,0)");
        
    // x axis
    var x = d3.scaleBand()
        .range([0, width])
        .domain(ng_data[0].map(d=>d.room_type))
        .padding(0.2);
    svg.append("g")
        .attr("transform","translate("+margin+","+(+width+margin)+")")
        .call(d3.axisBottom(x));
        
    // y axis
    var y = d3.scaleLinear()
        .domain([0,5000])
        .range([height, 0]);
    var y_axis = svg.append("g")
        .attr("class","y_axis");
    
    // x axis label
    svg.append("text")
      .attr("text-anchor", "end")      
      .attr("x", width - margin/2)
      .attr("y", height + margin + 50)
      .text("Room Type")

    // y axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-90)")
      .attr("x", -margin/2 - width/2)
      .attr("y", margin-50)
      .text("No. of Listings")
        
            
    // draw chart
    update_chart(ng_data[curr_ind]);
    update_chart(ng_data[curr_ind]);
    
    // update bar chart
    function update_chart(data){
        // update y axis
        var y = d3.scaleLinear()
            .domain([0,d3.max(data, function(d) { return d.count })])
            .range([height, 0]);
        y_axis.attr("transform","translate("+margin+","+margin+")")
            .transition().duration(1000)
            .call(d3.axisLeft(y));
    
        var chart = svg.selectAll("rect")
            .data(data);
    
        chart.enter()
            .append("rect")
            .merge(chart)
            .transition()
            .duration(1000)
                .attr("x", function(d) { return x(d.room_type); })
                .attr("y", function(d) { return y(d.count); })
                .attr("width", x.bandwidth())
                .attr("height", function(d) { return height - y(d.count); })
                .attr("transform","translate("+margin+","+margin+")")
                .attr("fill", "#69b3a2");
                
        // tooltip
        chart.on("mouseover", function(d){
                d3.select(this)
                    .style("stroke", "black");
                svg.append("text")
                     .attr("class", "val")
                     .attr("x", (d3.mouse(this)[0]+50) + "px")
                     .attr("y", (d3.mouse(this)[1]+80) + "px")
                     .style("fill", "#3a4c40")
                     .text(d.count + " listings");
            })
            .on("mouseleave", function(d){
                d3.select(this)
                    .style("stroke", "none");
                d3.selectAll(".val")
                    .remove();
            })
            .on("click", function(d){
                show_price_review(d.room_type); 
            });
    
    }


    // new scene
    var svg_list = d3.select("#div_list")
            .append("svg")
            .attr("width", width + margin + margin)
            .attr("height", height + margin + margin)
            .append("g")
            .attr("transform","translate(0,0)");
    
    var x_axis_plot = svg_list.append("g")
        .attr("class","x_axis_plot");
        
    var y_axis_plot = svg_list.append("g")
        .attr("class","y_axis_plot");
    
    // show scatterplot of price and reviews
    function show_price_review(room_type){           
        curr_nb = neighbourhood_group[curr_ind].values;

        var filtered_list = curr_nb.filter(function(d){ 
           if( d.room_type == room_type){ 
               return d;
           } 
        })

        d3.selectAll("#div_list > *").remove();
        svg_list = d3.select("#div_list")
            .append("svg")
            .attr("width", width + margin + margin)
            .attr("height", height + margin + margin)
            .append("g")
            .attr("transform","translate(0,0)");
        
        var x_axis_plot = svg_list.append("g")
            .attr("class","x_axis_plot");
        
        var y_axis_plot = svg_list.append("g")
            .attr("class","y_axis_plot");
        
        // x axis
        var x_plot = d3.scaleLinear()
            .domain([0,d3.max(filtered_list, function(d) { return d.number_of_reviews })])
            .range([0, width])
            .clamp(true);
        x_axis_plot.attr("transform","translate("+margin+","+(+width+margin)+")")
            .transition().duration(1000)
            .call(d3.axisBottom(x_plot));
        
        //y axis
        var y_plot = d3.scaleLinear()
            .domain([0,d3.max(filtered_list, function(d) { return d.price })])
            .range([height, 0])
            .clamp(true);
        y_axis_plot.attr("transform","translate("+margin+","+margin+")")
            .transition().duration(1000)
            .call(d3.axisLeft(y_plot));
        
        // x axis label
        svg_list.append("text")
          .attr("text-anchor", "end")      
          .attr("x", width - margin/2)
          .attr("y", height + margin + 50)
          .text("No. of Reviews")
    
        // y axis label
        svg_list.append("text")
          .attr("text-anchor", "end")
          .attr("transform", "rotate(-90)")
          .attr("x", -margin/2 - width/2)
          .attr("y", margin-50)
          .text("Price")
        
        // scatterplot
        svg_list.selectAll("circle")
            .data(filtered_list)
            .enter()
            .append("circle")
            .attr("transform","translate("+margin+","+margin+")")
            .attr("cx", function (d) { return x_plot(+d.number_of_reviews); } )
            .attr("cy", function (d) { return y_plot(+d.price); } )
            .attr("r", function (d) { return d.minimum_nights/10; } )
            .style("fill", "#69b3a2")
            .style("opacity", 0.65)
            .on("mouseover", function(d){
                svg_list.append("text")
                     .attr("class", "val2")
                     .attr("x", (d3.mouse(this)[0]+50) + "px")
                     .attr("y", (d3.mouse(this)[1]+80) + "px")
                     .style("fill", "#3a4c40")
                     .text(d.neighbourhood + " $" + d.price)
                 svg_list.append("text")
                     .attr("class", "val2")
                     .attr("x", (d3.mouse(this)[0]+50) + "px")
                     .attr("y", (d3.mouse(this)[1]+95) + "px")
                     .style("fill", "#3a4c40")
                     .text("Min. " + d.minimum_nights + " nights")
            })
            .on("mouseleave", function(d){
                d3.selectAll(".val2")
                    .remove()
            });
        
    }
}

// annotations
function draw_annotations(total, new_total){

    const annotations = [
        {
          note: {
            label: "out of "+total+" listings",
            title: new_total
          },
          x: 100,
          y: 0,
          dy: 0,
          dx: 0
        }].map(function(d){ d.color = "#E8336D"; return d});
        
    const makeAnnotations = d3.annotation()
          .type(d3.annotationLabel)
          .annotations(annotations);

    d3.selectAll("#listing > *").remove();
      
    svg = d3.select("#listing")
        .append("svg")
        .append("g")
        .attr("transform","translate(0,0)")
        .call(makeAnnotations);

}

</script>
 

<!--            
This is a normal paragraph following a header. GitHub is a code hosting platform for version control and collaboration. It lets you and others work together on projects from anywhere.

## Header 2

> This is a blockquote following a header.
>
> When something is important enough, you do it even if the odds are not in your favor.

### Header 3

```js
// Javascript code with syntax highlighting.
var fun = function lang(l) {
  dateformat.i18n = require('./lang/' + l)
  return true;
}
```

```ruby
# Ruby code with syntax highlighting
GitHubPages::Dependencies.gems.each do |gem, version|
  s.add_dependency(gem, "= #{version}")
end
```

#### Header 4

*   This is an unordered list following a header.
*   This is an unordered list following a header.
*   This is an unordered list following a header.

##### Header 5

1.  This is an ordered list following a header.
2.  This is an ordered list following a header.
3.  This is an ordered list following a header.

###### Header 6

| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| out of stock | good and plenty   | nice  |
| ok           | good `oreos`      | hmm   |
| ok           | good `zoute` drop | yumm  |

### There's a horizontal rule below this.

* * *

### Here is an unordered list:

*   Item foo
*   Item bar
*   Item baz
*   Item zip

### And an ordered list:

1.  Item one
1.  Item two
1.  Item three
1.  Item four

### And a nested list:

- level 1 item
  - level 2 item
  - level 2 item
    - level 3 item
    - level 3 item
- level 1 item
  - level 2 item
  - level 2 item
  - level 2 item
- level 1 item
  - level 2 item
  - level 2 item
- level 1 item

### Small image

![Octocat](https://github.githubassets.com/images/icons/emoji/octocat.png)

### Large image

![Branching](https://guides.github.com/activities/hello-world/branching.png)


### Definition lists can be used with HTML syntax.

<dl>
<dt>Name</dt>
<dd>Godzilla</dd>
<dt>Born</dt>
<dd>1952</dd>
<dt>Birthplace</dt>
<dd>Japan</dd>
<dt>Color</dt>
<dd>Green</dd>
</dl>

```
Long, single-line code blocks should not wrap. They should horizontally scroll if they are too long. This line should be long enough to demonstrate this.
```

```
The final element.
```
-->
