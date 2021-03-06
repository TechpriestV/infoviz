<template>
  <div>
    <div class="menuItems">
      <a href="javascript:history.go(-1)" class="route_button2">Back</a>
    </div>
    <div class="starCount">
      <div class="gameName">{{gameName}}</div>
    </div>

    <div class="dynamic-hover-details">
      <div class="staticHeadline">Viewing this game</div>
      <div class="changingValues">{{this.current.games[gameId].totalViewers}}</div>
    </div>
    <div class="cont">
      <div id="chart"></div>
      <Slider/>
    </div>
  </div>
</template>


<script>
import { mapState } from "vuex";
import Slider from "@/components/Slider.vue";
import * as d3 from "d3";
import gameColor from "../assets/js/colorsMixin.js";

export default {
  mixins: [gameColor],
  name: "ScatterPlot",
  data() {
    return {
      gameName: "",
      selectedStreamer: ""
    };
  },
  mounted() {
    this.initScatter();
    this.getGame();
  },
  computed: {
    gameId() {
      return this.$route.params.id;
    },
    streams() {
      const streams = this.current.games[this.gameId]
        ? this.current.games[this.gameId].streams
        : [];
      return streams;
    },
    ...mapState(["current", "games"])
  },
  components: {
    Slider
  },
  watch: {
    current() {
      this.initScatter();
    }
  },
  props: ["gameID"],
  methods: {
    getGame() {
      if (this.games[this.gameId]) {
        this.gameName = this.games[this.gameId].name;
        return true;
      } else {
        this.$store
          .dispatch("getGameInfo", this.gameId)
          .then(res => {
            this.gameName = res.name;
            return true;
          })
          .catch(error => {
            console.log(error);
          });
      }
    },
    initScatter() {
      const self = this;
      const streams = this.streams;

      //Scatterplot
      const margin = {
          left: document.documentElement.clientWidth * 0.03,
          top: document.documentElement.clientWidth * 0.04,
          right: document.documentElement.clientWidth * 0.05,
          bottom: document.documentElement.clientWidth * 0.04
        },
        width = document.documentElement.clientWidth / 1.3,
        height = document.documentElement.clientHeight / 1.4;

      d3.select("svg").remove();

      const svg = d3
        .select("#chart")
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom);

      const wrapper = svg
        .append("g")
        .attr("class", "chordWrapper")
        .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

      const tooltip = d3
        .select("#chart")
        .append("div")
        .attr("id", "tooltip")
        .style("position", "absolute")
        .style("display", "block")
        .style("visibility", "hidden")
        .style("color", "white")
        .style("background-color", "#242625")
        .style("opacity", 0.8)
        .style("border-radius", "8px")
        .style("font-family", "Lato")
        .style("text-align", "center")
        .style("padding", "2px")
        .style("word-wrap", "break-word")
        .style("max-width", "400px")
        .style("z-index", 1000)
        .text("");

      //////////////////////////////////////////////////////
      ///////////// Initialize Axes & Scales ///////////////
      //////////////////////////////////////////////////////

      const opacityCircles = 0.7;
      const maxDistanceFromPoint = 10;

      //Set the new x axis range
      const xScale = d3
        .scaleLog()
        .range([0, width])
        .domain(
          d3.extent(streams, function(d) {
            const view_count = d.view_count ? d.view_count : 0;
            return view_count;
          })
        )
        .nice();
      //Set new x-axis
      const xAxis = d3
        .axisBottom()
        .ticks(4)
        .tickFormat(function(d) {
          return xScale.tickFormat(4, function(d) {
            return d3.format(".2s")(d);
          })(d);
        })
        .scale(xScale);
      //Append the x-axis
      wrapper
        .append("g")
        .attr("class", "x axis")
        .attr("transform", "translate(" + 0 + "," + height + ")")
        .style("stroke", "white")
        .call(xAxis);

      //Set the new y axis range
      const yScale = d3
        .scaleLinear()
        .range([height, 0])
        .domain(
          d3.extent(streams, function(d) {
            const viewer_count = d.viewer_count ? d.viewer_count : 0;
            return viewer_count;
          })
        )
        .nice();
      const yAxis = d3
        .axisLeft()
        .ticks(6) //Set rough # of ticks
        .scale(yScale);
      //Append the y-axis
      wrapper
        .append("g")
        .attr("class", "y axis")
        .attr("transform", "translate(" + 0 + "," + 0 + ")")
        .style("stroke", "white")
        .call(yAxis);

      //////////////////////////////////////////////////////
      ///////////////// Initialize Labels //////////////////
      //////////////////////////////////////////////////////

      //Set up X axis label
      wrapper
        .append("g")
        .append("text")
        .attr("class", "x title")
        .attr("text-anchor", "end")
        .style("font-size", "12px")
        .style("font-family", "Lato")
        .style("font-weight", "400")
        .style("stroke", "white")
        .attr(
          "transform",
          "translate(" + (width + 15) + "," + (height - 10) + ")"
        )
        .text("Life time views");

      //Set up y axis label
      wrapper
        .append("g")
        .append("text")
        .attr("class", "y title")
        .attr("text-anchor", "end")
        .style("font-size", "12px")
        .style("font-family", "Lato")
        .style("font-weight", "400")
        .style("stroke", "white")
        .attr("transform", "translate(18, 0) rotate(-90)")
        .text("Current Viewers");

      ////////////////////////////////////////////////////////////
      ///// Capture mouse events and voronoi.find() the site /////
      ////////////////////////////////////////////////////////////

      // Use the same variables of the data in the .x and .y as used in the cx and cy of the circle call
      svg._tooltipped = svg._voronoi = null;
      svg.on("mousemove", function() {
        if (!svg._voronoi) {
          svg._voronoi = d3
            .voronoi()
            .x(function(d) {
              return xScale(d.view_count);
            })
            .y(function(d) {
              return yScale(d.viewer_count);
            })(streams);
        }
        var p = d3.mouse(this),
          site;
        p[0] -= margin.left;
        p[1] -= margin.top;
        // don't react if the mouse is close to one of the axis
        if (p[0] < 1 || p[1] < 1) {
          site = null;
        } else {
          site = svg._voronoi.find(p[0], p[1], maxDistanceFromPoint);
        }
        if (site !== svg._tooltipped) {
          if (svg._tooltipped) removeTooltip(svg._tooltipped.data);
          if (site) showTooltip(site.data);
          svg._tooltipped = site;
        }
        if (d3.event.pageY < document.documentElement.clientHeight / 2) {
          tooltip.style("top", d3.event.pageY - 120 + "px");
        } else {
          tooltip.style("top", d3.event.pageY - 220 + "px");
        }

        if (d3.event.pageX < document.documentElement.clientWidth / 2) {
          tooltip.style("left", d3.event.pageX + 10 + "px");
        } else {
          tooltip.style("left", d3.event.pageX - 400 + "px");
        }

        return tooltip;
      });

      ////////////////////////////////////////////////////////////
      /////////////////// Scatterplot Circles ////////////////////
      ////////////////////////////////////////////////////////////

      //Initiate a group element for the circles
      const circleGroup = wrapper
        .append("g")
        .attr("class", "circleWrapper")
        .selectAll("streamer")
        .data(streams);

      circleGroup.exit().remove();
      //Place the country circles
      circleGroup
        .enter()
        .append("circle")
        .merge(circleGroup)
        .attr("class", function(d, i) {
          return "streamer a" + d.display_name;
        })
        .attr("cx", function(d) {
          return xScale(d.view_count);
        })
        .attr("cy", function(d) {
          return yScale(d.viewer_count);
        })
        .on("click", d => {
          this.selectedStreamer = d.id;
        })
        .attr("r", "8")
        .style("opacity", opacityCircles)
        .style("fill", d => {
          return this.gameColor(d.game_id);
        });

      ///////////////////////////////////////////////////////////////////////////
      /////////////////// Hover functions of the circles ////////////////////////
      ///////////////////////////////////////////////////////////////////////////

      //Hide the tooltip when the mouse moves away
      function removeTooltip(d, i) {
        //Save the chosen circle (so not the voronoi)
        const element = d3.selectAll(".streamer.a" + d.display_name);

        //Fade out the bubble again
        element.style("opacity", opacityCircles);

        //Hide tooltip
        d3.select(".popover").each(function() {
          d3.select(this).style("visibility", "hidden");
        });

        //Fade out guide lines, then remove them
        d3.selectAll(".guide")
          .transition()
          .duration(100)
          .style("opacity", 0)
          .remove();

        return tooltip.style("visibility", "hidden").style("display", "none");
      } //function removeTooltip

      //Show the tooltip on the hovered over slice
      function showTooltip(d, i) {
        //Save the chosen circle (so not the voronoi)
        const element = d3.select(".streamer.a" + d.display_name),
          el = element._groups[0];
        if (d.offline_image_url == "") {
          d.offline_image_url =
            "https://static-cdn.jtvnw.net/ttv-boxart/404_boxart-80x112.jpg";
        }
        tooltip.on("click", () => {
          self.selectedStreamer = d.id;
        });
        tooltip.html(
          '<h2 id="zoom_tooltip">' +
            d.display_name +
            "</h2>" +
            "<img src=" +
            d.offline_image_url +
            ' style="display:inline-block;max-width:200;max-height:170px;width:auto;height:auto;padding:10px;"/>' +
            '<p id="p_tooltip">' +
            d.title +
            "</p>"
        );

        //Make chosen circle more visible
        element.style("opacity", 1);

        //Place and show tooltip
        const x = +element.attr("cx"),
          y = +element.attr("cy"),
          color = element.style("fill");

        //Append lines to bubbles that will be used to show the precise data points

        //vertical line
        wrapper
          .append("line")
          .attr("class", "guide")
          .attr("x1", x)
          .attr("x2", x)
          .attr("y1", y)
          .attr("y2", height + 20)
          .style("stroke", color)
          .style("opacity", 0)
          .transition()
          .duration(100)
          .style("opacity", 0.5);
        //Value on the axis
        wrapper
          .append("text")
          .attr("class", "guide")
          .attr("x", x)
          .attr("y", height + 38)
          .style("fill", color)
          .style("opacity", 0)
          .style("text-anchor", "middle")
          .text(d3.format(".2s")(d.view_count))
          .transition()
          .duration(100)
          .style("opacity", 0.5);

        //horizontal line
        wrapper
          .append("line")
          .attr("class", "guide")
          .attr("x1", x)
          .attr("x2", "1")
          .attr("y1", y)
          .attr("y2", y)
          .style("stroke", color)
          .style("opacity", 0)
          .transition()
          .duration(100)
          .style("opacity", 0.5);
        //Value on the axis
        wrapper
          .append("text")
          .attr("class", "guide")
          .attr("x", "1")
          .attr("y", y)
          .attr("dy", "0.35em")
          .style("fill", color)
          .style("opacity", 0)
          .style("text-anchor", "end")
          .text(d.viewer_count)
          .transition()
          .duration(100)
          .style("opacity", 0.5);

        return tooltip.style("visibility", "visible").style("display", "block");
      } //function showTooltip
    }
  }
};
</script>
<style scoped lang="scss">
.gameName {
  width: 100vw;
  margin: 0;
  justify-content: center;
  text-transform: uppercase;
  font-family: Lato;
  font-weight: 400;
  color: white;
  font-size: 1.6vw;
  letter-spacing: 2px;
}
.starCount {
  float: left;
  margin-top: -5%;
}
.dynamic-hover-details {
  position: absolute;
  right: 0px;
  top: 30%;
  flex-direction: column;
  float: left;
  text-transform: uppercase;
  width: 15%;
  display: flex;
  // letter-spacing: 2px;
  // font-size: 0.8vw;
}
.static-headline {
  width: 100%;
  display: block;
  color: white;
  font-family: Lato;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 2px;
  font-size: 0.8vw;
}

.changingValues {
  color: #e81b5f;
  font-family: Lato;
  font-weight: 300;
  font-size: 20px;
}

.axis path,
.axis line {
  fill: none;
  stroke: #b3b3b3;
  shape-rendering: crispEdges;
}
.axis text {
  font-size: 10px;
  fill: #6b6b6b;
}

.guide {
  pointer-events: none;
  font-size: 14px;
  font-weight: 600;
}

.popover {
  position: absolute;
  pointer-events: none;
}
</style>
