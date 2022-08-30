<template>
  <div
    class="
      container
      mx-auto
      py-4
      h-screen
      flex flex-col
      gap-y-4
    "
  >
    <div class="items-center justify-center">
      <label for="rotation" class="block mb-2 text-sm font-bold text-gray-800">Rotate</label>
      <input id="rotation" min="0" max="360" type="range" @input="draw()" v-model="rotationDegrees" class="w-1/4 h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer">
    </div>

    <div class="flex flex-row items-center justify-center gap-x-4">
      <div class="flex flex-col gap-y-4">
        <image-uploader @change="imageFile" />
        <button class="font-bold p-2 border-2 cursor-pointer hover:bg-gray-200" @click="addText">
          Add Text
        </button>
      </div>
      <canvas
        id="myCanvas"
        width="800"
        height="450"
        class="border-2 border-gray-800"
        @mousedown="handleMouseDown"
        @mouseup="handleMouseUp"
        @mousemove="handleMouseMove"
        @mouseout="handleMouseOut"
      >
      </canvas>
    </div>
  </div>
</template>

<script>
// @ is an alias to /src
import ImageUploader from "@/components/ImageUploader.vue";
// import { ref } from 'vue'

export default {
  name: "HomeView",
  components: {
    ImageUploader,
  },
  data() {
    return {
      elements: [],
      startX: null,
      startY: null,
      selected: -1,
      canvas: null,
      context: null,
      pi2: Math.PI * 2,
      resizerRadius: 8,
      rotationDegrees: 0,
      // draggingResizer: -1
    };
  },
  mounted() {
    this.canvas = document.getElementById("myCanvas");
    this.context = this.canvas.getContext("2d");    // two-dimensional rendering context
  },
  methods: {
    async imageFile() {
      let img = document.querySelector(".dropzoneFile").files[0];
      let bitmap = await createImageBitmap(img);
      this.elements.push({
        type: "image",
        content: bitmap,
        x: 0,
        y: 20,
        width: bitmap.width, //calculate width & height
        height: bitmap.height,
        isSelected: false,
        isDragging: false,
        rotate: 45
      });
      console.log("width", bitmap.width);
      console.log("height", bitmap.height);
      this.draw();
    },
    addText() {
      this.context.font = "30px Arial";
      this.elements.push({
        type: "text",
        content: "Your Text Here",
        x: Math.random() * 500,
        y: Math.random() * 400,
        width: this.context.measureText("Your Text Here").width, //calculate width & height
        height: 30,
        isSelected: false,  
        isDragging: false,
        draggingResizer: -1
      });
      //draw
      this.draw();
    },
    degreesToRadian(d) {
      return d * 0.01745;   // One degree is equal to 0.0174533 radians.
    },
    draw() {
      /*####################################################

      UPDATE: 
        > able to rotate the image and text using the slider
        > text can be resized but not optimized

      FIXED:
        > able to upload the same image file

      ####################################################*/

      this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
      for (var i = 0; i < this.elements.length; i++) {
          var object = this.elements[i];
          
          this.context.save();
          if(object.isSelected || object.isDragging) {
            this.context.translate(object.x + (object.width/2), object.y + (object.height/2));
            // this.context.rotate(this.rotationDegrees*Math.PI/180.0);
            this.context.rotate(this.degreesToRadian(this.rotationDegrees));
            //after translating, unstranslate
            this.context.translate(-object.x - (object.width/2), -object.y - (object.height/2));    // the magic line

            let imageRight = object.x + object.width;
            let imageBottom = object.type == 'text' ? (object.y - object.height) : (object.y + object.height);
            this.drawDragAnchor(object.x, object.y);     // top-left
            this.drawDragAnchor(imageRight, object.y);    // top-right
            this.drawDragAnchor(imageRight, imageBottom);  // bottom-left
            this.drawDragAnchor(object.x, imageBottom);    // top-right

            this.context.beginPath();
            this.context.moveTo(object.x, object.y);
            this.context.lineTo(imageRight, object.y);
            this.context.lineTo(imageRight, imageBottom);
            this.context.lineTo(object.x, imageBottom);
            this.context.closePath();
            this.context.stroke();
          }

          if(object.type == 'text') {
            this.context.font = `${object.height}px Arial` ;
            this.context.fillText(object.content, object.x, object.y, object.width, object.height);
          }
          else {
            this.context.drawImage(object.content, object.x, object.y, object.width, object.height);
          }
          
          this.context.restore();
      }
    },
    drawDragAnchor(x, y) {
      // this.context.fillRect(x, y, 10, 10);
      this.context.beginPath();
      // arc begins at an angle of 0 radians (0°), and ends at an angle of 2π radians (360°)
      this.context.arc(x, y, this.resizerRadius, 0, this.pi2, false);
      this.context.closePath();
      this.context.fill();
    },
    anchorClicked(x, y, i) {
      let dx, dy;

      let object = this.elements[i];
      var rr = this.resizerRadius + this.resizerRadius;
      let imageRight = object.x + object.width;
      let imageBottom = object.type == 'text' ? (object.y - object.height) : (object.y + object.height);
       
      console.log(x,y, i);
      // top-left
      dx = x - object.x;
      dy = y - object.y;
      if (dx * dx + dy * dy <= rr) {
          return (0);
      }
      // top-right
      dx = x - imageRight;
      dy = y - object.y;
      if (dx * dx + dy * dy <= rr) {
          return (1);
      }
      // bottom-right
      dx = x - imageRight;
      dy = y - imageBottom;
      if (dx * dx + dy * dy <= rr) {
          return (2);
      }
      // bottom-left
      dx = x - object.x;
      dy = y - imageBottom;
      if (dx * dx + dy * dy <= rr) {
          return (3);
      }
      return (-1);
    },
    objectBoundaries(cursorX, cursorY, index) {
      let object = this.elements[index];
      console.log(cursorY, object.y);
      if (object.type == 'image' && cursorX >= object.x && cursorX <= object.x + object.width && 
        cursorY >= object.y && cursorY <= object.y + object.height) {
        return true;
      }
      if (object.type == 'text' && cursorX >= object.x && cursorX <= object.x + object.width && 
        cursorY >= object.y - object.height && cursorY <= object.y) {
        return true;
      }
      return false;
    },
    handleMouseDown(e) {
      // handle mousedown events
      // iterate through elements[] and see if the user
      // mousedown'ed on one of them
      // If yes, set the selected Text to the index of that text
      e.preventDefault(); //stops the default action of a selected element from happening by a user.
      let canvasOffset = $("#myCanvas").offset();
      this.startX = parseInt(e.clientX - canvasOffset.left);  //-offsetX
      this.startY = parseInt(e.clientY - canvasOffset.top);
      // console.log(e.clientX, e.clientY);
      // console.log(this.startX, this.startY);
      
      for (var i = 0; i < this.elements.length; i++) {
        this.elements[i].draggingResizer = this.anchorClicked(this.startX, this.startY, i);
        if(this.elements[i].draggingResizer > -1) {
          this.selected = i;
          this.elements[i].isDragging = true;
          this.elements[i].isSelected = true;
        }
        else if (this.objectBoundaries(this.startX, this.startY, i)) {
          this.selected = i;
          this.elements[i].isSelected = true;
          this.elements[i].isDragging = false;
        }
        else {
          this.elements[i].isSelected = false;
          this.elements[i].isDragging = false;
        }
      }
    },
    handleMouseMove(e) {
      if (this.selected < 0) return;
      
      e.preventDefault();
      let canvasOffset = $("#myCanvas").offset();
      let mouseX = parseInt(e.clientX - canvasOffset.left);
      let mouseY = parseInt(e.clientY - canvasOffset.top);
      let currentObject = this.elements[this.selected];
      let imageRight = currentObject.x + currentObject.width;
      let imageBottom = currentObject.type == 'text' ? (currentObject.y - currentObject.height) : (currentObject.y + currentObject.height);
      
      // move the image by the amount of the latest drag
      if(currentObject.draggingResizer > -1) {
        // resize the image 
        switch (currentObject.draggingResizer) {
          case 0:
              //top-left
              currentObject.x = mouseX;       // changed this.startX and this.startY to mouseX and mouseY
              currentObject.width = imageRight - mouseX;   
              currentObject.y = mouseY;
              currentObject.height = imageBottom - mouseY;
              console.log(currentObject.width, imageRight, mouseX, imageRight - mouseX);
              break;
          case 1:
              //top-right
              currentObject.y = mouseY;
              currentObject.width = mouseX - currentObject.x;
              currentObject.height = imageBottom - mouseY;
              break;
          case 2:
              //bottom-right
              currentObject.width = mouseX - currentObject.x;
              currentObject.height = mouseY - currentObject.y;
              break;
          case 3:
              //bottom-left
              currentObject.x = mouseX;
              currentObject.width = imageRight - mouseX;
              currentObject.height = mouseY - currentObject.y;
              break;
        }

        // if(currentObject.width<25){currentObject.width=25;}
        // if(currentObject.height<25){currentObject.height=25;}

        // set the image right and bottom
        // imageRight = this.startX + currentObject.width;
        // imageBottom = this.startY + currentObject.height;
        // this.context.drawImage(currentObject.content, currentObject.x, currentObject.y, currentObject.width, currentObject.height);
        
      }
      else if(this.elements[this.selected].isSelected) {
        var dx = mouseX - this.startX;
        var dy = mouseY - this.startY;
        // reset the startXY for next time
        this.startX = mouseX;
        this.startY = mouseY;

        var object = this.elements[this.selected];
        object.x += dx;
        object.y += dy;
      }
      this.draw();
    },
    handleMouseUp(e) {
      e.preventDefault();
      this.selected = -1;   //not holding the object
      this.elements.forEach(element => {
        element.isDragging = false;
        element.draggingResizer = -1;
      });
      this.draw();
    },
    handleMouseOut(e) {
      this.handleMouseUp(e);
    },
  },
};
</script>



