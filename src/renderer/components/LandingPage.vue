<template>
  <el-container>
    <!-- <el-header>
      <h1>逆地理编码</h1>
    </el-header> -->
    <el-main>
      <el-row :gutter="20">
        <el-col :span="16">
          <el-input v-model="readPath" disabled></el-input>
        </el-col>
        <el-col :span="8">
          <el-button @click="open" :disabled="isloading">上传</el-button>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="16">
          <el-input v-model="writePath" disabled></el-input>
        </el-col>
        <el-col :span="4">
          <el-button @click="output" :disabled="isloading">输出路径</el-button>
        </el-col>
         <el-col :span="4">
          <el-button @click="execute" :disabled="isloading">开始导出</el-button>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="16">
          <el-progress :text-inside="true" :stroke-width="18" :percentage="currPercent"></el-progress>
        </el-col>
         <el-col :span="4">
           <span>{{curr+'/'+total}}</span>
        </el-col>
      </el-row>
      
    </el-main>
  </el-container>
</template>

<script>
  import SystemInformation from './LandingPage/SystemInformation'
  import csv from 'csv-parse'
  import fs from 'fs'
  import async from 'async'
  // import lineReader from 'line-reader'
  export default {
    name: 'landing-page',
    components: { SystemInformation },
    data() {
      return {
        csv: null,
        batchNumber: 10,
        readPath: "",
        writePath: "",
        outpath: "",
        fileName: "",
        writeStream: null,
        headers: ["location","formatted_address", "level"],
        currPercent: 0,
        total: 0,
        curr: 0,
        isloading: false
      }
    },
    created() {
      // this.csv = csv.createParser();
      var os=require('os');
      var homedir=os.homedir();
      this.writePath = this.outpath = homedir + "\\Desktop";
      console.log('homedir', homedir);
       //this.$electron.ipcRenderer.on('selectedItem',this.getPath);
    },
    methods: {
      open (link) {
        this.$electron.remote.dialog.showOpenDialog({
          properties: ['multiSelections'],
          message: '需要导入的文件',
          filters: [
                    { name: 'Excel', extensions: ['csv', 'elx', 'gif'] },
                    { name: 'Movies', extensions: ['mkv', 'avi', 'mp4'] },
                    { name: 'Custom File Type', extensions: ['as'] },
                    { name: 'All Files', extensions: ['*'] }
                  ]
          }, (path) => {
            if(!path){ return;}
            this.readPath = path[0];
            let _t = this.readPath.split("\\");
            this.fileName = _t[_t.length-1];
            this.writePath =  this.outpath + "\\逆地理编码导出-" + this.fileName;
            // this.getPath(path[0])
          })
        // this.$electron.ipcRenderer.send('open-directory-dialog','openDirectory');
      },
      reset() {
        this.currPercent = 0;
        this.total = 0;
        this.curr = 0;
        this.isloading = false;
      },
      output (link) {
        
        this.$electron.remote.dialog.showOpenDialog({
          properties: ['openDirectory'],
          message: '需要导入的文件',
          filters: [
                    { name: 'Excel', extensions: ['csv', 'elx', 'gif'] },
                    { name: 'Movies', extensions: ['mkv', 'avi', 'mp4'] },
                    { name: 'Custom File Type', extensions: ['as'] },
                    { name: 'All Files', extensions: ['*'] }
                  ]
          }, (path) => {
            if(!path){
              return;
            }
            this.outpath = path;
            this.writePath = path + "\\逆地理编码导出-" + this.fileName;
            // this.getPath(path[0])
          })
        // this.$electron.ipcRenderer.send('open-directory-dialog','openDirectory');
      },
      /**
       * 开始导出
       */
      getPath(path) {
        let count = 0;
        let cacheList = [];
        const self = this;
        const batchNumber = this.batchNumber;
        self.total=0, self.curr=0;
        self.currPercent = 0;
        let isLast = false;
        self.isloading = true;
       var parser = csv({delimiter: ','}, function (err, data) {
         self.total=data.length;
          async.eachSeries(data, function (line, next) {
            // console.log(line);
            // if(line[0] === 'CODE'){next(); return;}
            // console.log(count, curr);
            self.curr++;
            self.currPercent = Math.floor(self.curr/self.total * 100);
            if(self.curr >= self.total){
              // console.log('最后一个');
              isLast = true;
            }
            cacheList.push(line);
            if(++count > batchNumber-1 || isLast){
              self.getGeo(cacheList).then(res => {
                console.log(res);
                cacheList.forEach((item, index) => {
                  let str = "";
                  str+=cacheList[index][0] + ",";
                  str+=cacheList[index][1] + ",";
                  const _data = res.data.geocodes[index];
                  self.headers.forEach(header => {
                    str+=_data[header]+","
                  });
                  self.writeStream.write(str+"\n");
                })
                cacheList.length = 0;
                count = 0;
                if(isLast){
                  self.writeStream.close();
                  self.$message({
                    message: '导出成功！',
                    type: 'success',
                    duration: 10000,
                    showClose: true
                  });
                  let myNotification = new Notification('导出成功！', {
                    body: '导出成功！'
                  });
                  self.isloading = false;
                }
                next();
              }).catch(err => {
                console.error(err);
                self.isloading = false;
              })
            }else{
              next();
            }
            
            // do something with the line
            // doSomething(line).then(function() {
            //   // when processing finishes invoke the callback to move to the next one
            //   callback();
            // });
          })
        });
        fs.createReadStream(path).pipe(parser);
      },
      getGeo(lineDataList) {
        let addrStr = ""
        lineDataList.forEach(item => {
          addrStr += item[1] + "|"
        })
        let params = {
          key: "d81c1d2dfcf69ab34511bc84709ab556",
          address: addrStr,
          city: "天津",
          batch: true,
          output: "JSON",
        }
        return this.$http.get('https://restapi.amap.com/v3/geocode/geo?key=d81c1d2dfcf69ab34511bc84709ab556&city=天津&batch=true&address='+addrStr)
      },
      getWriteStream(path) {
        
        return fs.createWriteStream(this.writePath,{encoding:'utf8'});
      },
      execute() {
        if(!this.readPath){
          this.$message({
            message: '请选择文件！',
            type: 'error'
          })
          return;
        }
        this.writeStream = this.getWriteStream();
        this.writeStream.write('CODE,NAME,lon,lat,formatted_address,level\n');
        this.getPath(this.readPath);
        // writeStream.write('这是我要做的测试内容,');
        // writeStream.write('这是我要做的测试内容2');
        // writeStream.end();
      },
      writeLine(obj) {

      }
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
   .el-row {
    margin-bottom: 20px;
  }
  body { font-family: 'Source Sans Pro', sans-serif; }

  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    padding: 60px 80px;
    width: 100vw;
  }

  #logo {
    height: auto;
    margin-bottom: 20px;
    width: 420px;
  }

  main {
    display: flex;
    justify-content: space-between;
  }

  main > div { flex-basis: 50%; }

  .left-side {
    display: flex;
    flex-direction: column;
  }

  .welcome {
    color: #555;
    font-size: 23px;
    margin-bottom: 10px;
  }

  .title {
    color: #2c3e50;
    font-size: 20px;
    font-weight: bold;
    margin-bottom: 6px;
  }

  .title.alt {
    font-size: 18px;
    margin-bottom: 10px;
  }

  .doc p {
    color: black;
    margin-bottom: 10px;
  }

  .doc button {
    font-size: .8em;
    cursor: pointer;
    outline: none;
    padding: 0.75em 2em;
    border-radius: 2em;
    display: inline-block;
    color: #fff;
    background-color: #4fc08d;
    transition: all 0.15s ease;
    box-sizing: border-box;
    border: 1px solid #4fc08d;
  }

  .doc button.alt {
    color: #42b983;
    background-color: transparent;
  }
</style>
