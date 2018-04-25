<template>
  <div id="app">
    <Form class="form"  label-position="right" :label-width="100">
      <FormItem class="text-left" label="操作">
        <Button type="primary" @click="upload"><input accept="image/*" v-show="false" ref="input" type="file" @change="change" multiple/>选择图片</Button>
        <Button type="primary" @click="download">压缩</Button>
        <!-- <Button type="primary" @click="downloadLueSuo">生成略缩图</Button>         -->
      </FormItem>
      <!-- <FormItem label="略缩图大小">
        <Input class="lue-size" size="small" v-model="lx" placeholder="宽" />
        <Input class="lue-size" size="small" v-model="ly" placeholder="高" />        
      </FormItem> -->
      <FormItem label="压缩比">
         <Slider class="slider" v-model="ratio"/>
      </FormItem>
      <FormItem label="图片列表" v-if="files.length > 0">
        <ul class="file-list">
          <li v-for="(f, index) in files" :key="index">
            <span class="name">{{ f.name }}</span>
            &nbsp;&nbsp;&nbsp;&nbsp;
            <span class="size">{{ f.size / 1024 }}k</span>
          </li>
        </ul>
      </FormItem>
    </Form>

    <Card v-show="compressed" v-for="(f, index) in compressedImage" :key="index" class="cavnas">
      <div>
        <canvas :ref="index"></canvas>
        <div>
          <span class="name">{{f.name}}</span>
          &nbsp;&nbsp;&nbsp;
          <span class="size">尺寸:{{f.size}}k</span>
          &nbsp;&nbsp;&nbsp;
          <span class="reduceSize">减少尺寸:{{f.reduceSize}}k</span>
        </div>
      </div>
    </Card>
  </div>
</template>

<script>
import { Button, Slider, Card, Form, FormItem, LoadingBar, Input } from "iview";
import JSZip from "JSZip";
import FileSaver from 'file-saver';
import { constants } from 'http2';
export default {
  name: "App",
  components: {
    Button,
    Slider,
    Card,
    Form,
    FormItem,
    Input
  },
  data() {
    return {
      files: {},
      ratio: 50,
      zip: new JSZip(),
      compressedImage: [],
      compressed: false,
      lueSuo: false,
      lx: 0,
      ly: 0,
    };
  },
  methods: {
    upload() {
      this.$refs.input.click();
    },
    change(e) {
      this.files = e.target.files;
      this.$nextTick(() => {
        this.compressedImage = Array.from({ length: this.files.length }).map(() => ({name:'', size:0, reduceSize:0}));
        this.compressed = false;
      });
    },
    downloadLueSuo() {
      this.lueSuo = true;
    },
    read(file, index) {
      if(!file.type) {
        return
      }
      const reader = new FileReader();
      reader.readAsDataURL(file);
      const canvas = this.$refs[index][0];
      const context = canvas.getContext("2d");
      const ratio = (100 - this.ratio) / 100;
      const img = new Image();
      return new Promise(resolve => {
        reader.onload = e => {
          img.src = e.target.result;
          img.onload = () => {
            canvas.width = img.width;
            canvas.height = img.height;
            // if(this.lueSuo) {
            //   // context.drawImage(img, 0, 0, img.width, img.height, ratio);
            // } else {
            // }
            context.drawImage(img, 0, 0, img.width, img.height);
            console.log(ratio)
            canvas.toBlob(
              blob => {
                const reduceSize = ((file.size - blob.size) / 1024).toFixed(2);
                resolve({ blob, name: file.name, size: (blob.size/1024).toFixed(2), reduceSize });
              },
              "image/jpeg",
              ratio
            );
          };
        };
      });
    },
    download() {
      LoadingBar.start()
      this.$nextTick(async() => {
        const all = [];        
        for(let key in this.files) {
          all.push(this.read(this.files[key], key));
        }
        const images = await Promise.all(all);
        const zipfolder = this.zip.folder('mini-images');
        images.forEach((img, index) => {
          if(!img) {
            return
          }
          zipfolder.file(img.name, img.blob);
          this.compressedImage.splice(index, 1, { name: img.name, size: img.size, reduceSize: img.reduceSize });
          LoadingBar.finish();
          
        })
        zipfolder.generateAsync({type:"blob"})
        .then(content => {
            this.compressed = true;
            FileSaver.saveAs(content, "example.zip");
            this.files = {};
            this.lueSuo = false;
        });
      });
    }
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}
.slider-box {
  display: flex;
  width: 300px;
  margin: 0 auto;
}
.slider {
  width: 300px;
}
.form {
  width: 500px;
}
.text-left {
  text-align: left;
}
.cavnas {
  padding: 5px;
}
canvas {
  width: 300px;
  margin: 5px;
}
.file-list {
  margin: 0;
}
.file-list li {
  list-style: none;
}
.name {
  color: #999;
}
.size {
  color: #5cb85c;
}
.reduceSize {
  color: #ed3f14;
}
.lue-size {
  display: inline-block;
  width: 20%;
  margin-right: 5px;
}
</style>
