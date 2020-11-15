<template>
    <div id="app">
        <div class="upload-file">
            <h2>Загрузите файл с данными для построения графика</h2>
            <input @change="loadFile" type="file">
            <p class="error" v-if="showErrorMsg">{{errorMsg}}</p>
            <Plotter v-if="showPlotter" :plotterData="plotterData"/>
        </div>
        <div class="coordinates">
            <h2 v-if="plotterData.transmitterRoute.length > 0">Координаты</h2>
            <p v-for="(route,index) in plotterData.transmitterRoute" :key="index">
                {{index+1}}: ({{route.x}};{{route.y}})
            </p>
        </div>
    </div>
</template>

<script>
  import Plotter from './components/Plotter.vue'

  export default {
    name: 'App',
    components: {
      Plotter
    },
    data() {
      return {
        errorMsg: '',
        showPlotter: false,
        showErrorMsg: false,
        plotterData: {
          receiverA: {x: 0, y: 0},
          receiverB: {x: 0, y: 0},
          receiverC: {x: 0, y: 0},
          transmitterRoute: [],
          minCoords: {x: 0, y: 0},
          maxCoords: {x: 0, y: 0},
        }
      }
    },
    methods: {
      loadFile(event) {
        this.showErrorMsg = false;
        this.showPlotter = false;
        const file = event.target.files[0];
        const type = file.type.replace(/\/.+/, '')
        if (type === 'text') {
          const reader = new FileReader()
          reader.readAsText(file)
          reader.onload = () => {
            const result = reader.result.split('\n');
            const receiversPoint = result[0].split(',').map(num => Number(num));
            if (receiversPoint.some(point => isNaN(point))) {
              this.errorMsg = 'Некорректные данные в файле'
              this.showErrorMsg = true;
              return
            }
            const receiversRadiusList = result.filter((item, index) => index !== 0)
              .map(item => item.split(',').map(num => Number(num)));
            if (receiversRadiusList.some(receiversRadius => receiversRadius.some(radius => isNaN(radius)))) {
              this.errorMsg = 'Некорректные данные в файле'
              this.showErrorMsg = true;
              return
            }
            const receiverA = {
              x: receiversPoint[0],
              y: receiversPoint[1]
            }
            this.plotterData.receiverA = receiverA
            const receiverB = {
              x: receiversPoint[2],
              y: receiversPoint[3]
            }
            this.plotterData.receiverB = receiverB
            const receiverC = {
              x: receiversPoint[4],
              y: receiversPoint[5]
            }
            this.plotterData.receiverC = receiverC
            let transmitterRoute = []
            const minCoords = {
              x: Math.min(receiverA.x, receiverB.x, receiverC.x),
              y: Math.min(receiverA.y, receiverB.y, receiverC.y),
            }
            const maxCoords = {
              x: Math.max(receiverA.x, receiverB.x, receiverC.x),
              y: Math.max(receiverA.y, receiverB.y, receiverC.y),
            }
            receiversRadiusList.forEach(radiusArr => {
              receiverA.r = radiusArr[0];
              receiverB.r = radiusArr[1];
              receiverC.r = radiusArr[2];
              const transmitterPoint = this.findTransmitter(receiverA, receiverB, receiverC)
              minCoords.x = Math.min(transmitterPoint.x, minCoords.x)
              minCoords.y = Math.min(transmitterPoint.y, minCoords.y)
              maxCoords.x = Math.max(transmitterPoint.x, maxCoords.x)
              maxCoords.y = Math.max(transmitterPoint.y, maxCoords.y)
              transmitterRoute.push(transmitterPoint)
            })
            this.plotterData.transmitterRoute = transmitterRoute
            this.plotterData.minCoords = minCoords
            this.plotterData.maxCoords = maxCoords
            this.showPlotter = true;
          }
          reader.onerror = () => {
            this.errorMsg = 'Ошибка при чтении файла'
            this.showErrorMsg = true;
          }
        } else {
          this.errorMsg = 'Неверный формат файла'
          this.showErrorMsg = true;
        }
      },
      findTransmitter(receiverA, receiverB, receiverC) {
        const trianglePoints = {};
        const ABReceiversCrossing = this.findPoints(receiverA, receiverB);
        const BCReceiversCrossing = this.findPoints(receiverB, receiverC);
        const ACReceiversCrossing = this.findPoints(receiverA, receiverC);
        let distances = [{
          point1: ABReceiversCrossing.point1,
          point2: BCReceiversCrossing.point1,
          distance: this.getPointsDistance(ABReceiversCrossing.point1, BCReceiversCrossing.point1)
        }]
        if (BCReceiversCrossing.twoPoints) {
          distances.push({
            point1: ABReceiversCrossing.point1,
            point2: BCReceiversCrossing.point2,
            distance: this.getPointsDistance(ABReceiversCrossing.point1, BCReceiversCrossing.point2)
          })
        }
        if (ABReceiversCrossing.twoPoints) {
          distances.push({
            point1: ABReceiversCrossing.point2,
            point2: BCReceiversCrossing.point1,
            distance: this.getPointsDistance(ABReceiversCrossing.point2, BCReceiversCrossing.point1)
          })
        }
        if (ABReceiversCrossing.twoPoints && BCReceiversCrossing.twoPoints) {
          distances.push({
            point1: ABReceiversCrossing.point2,
            point2: BCReceiversCrossing.point2,
            distance: this.getPointsDistance(ABReceiversCrossing.point2, BCReceiversCrossing.point2)
          })
        }
        const minDistance = distances.reduce((acc, item) => item.distance < acc.distance ? item : acc);
        trianglePoints.firstVertex = minDistance.point1;
        trianglePoints.secondVertex = minDistance.point2;
        if (ACReceiversCrossing.twoPoints) {
          const firstDistance = this.getPointsDistance(trianglePoints.firstVertex, ACReceiversCrossing.point1);
          const secondDistance = this.getPointsDistance(trianglePoints.firstVertex, ACReceiversCrossing.point2);
          trianglePoints.thirdVertex = firstDistance < secondDistance ?
            ACReceiversCrossing.point1 : ACReceiversCrossing.point2;
        } else {
          trianglePoints.thirdVertex = ACReceiversCrossing.point1;
        }
        return this.getTriangleCentroid(trianglePoints);
      },
      findPoints(firstReceiver, secondReceiver) {
        const receiversDistance = this.getPointsDistance(firstReceiver, secondReceiver);

        //радиусы не касаются друг друга
        if (receiversDistance > firstReceiver.r + secondReceiver.r) {
          const receiverToPointDistance =
            firstReceiver.r + (receiversDistance - (firstReceiver.r + secondReceiver.r)) / 2;
          const point1 = this.getPointCoords(
            firstReceiver,
            secondReceiver,
            receiverToPointDistance,
            receiversDistance
          );
          return ({
            twoPoints: false,
            point1,
          });
        }

        //один радиус внутри другого
        if (receiversDistance < Math.abs(firstReceiver.r - secondReceiver.r)) {
          const largerRadiusReceiver = firstReceiver.r > secondReceiver.r ? firstReceiver : secondReceiver
          const smallerRadiusReceiver = firstReceiver.r < secondReceiver.r ? firstReceiver : secondReceiver
          const receiverToPointDistance =
            largerRadiusReceiver.r - (largerRadiusReceiver.r - (receiversDistance + smallerRadiusReceiver.r)) / 2;
          const point1 = this.getPointCoords(
            largerRadiusReceiver,
            smallerRadiusReceiver,
            receiverToPointDistance,
            receiversDistance
          );
          return ({
            twoPoints: false,
            point1,
          });
        }

        // радиусы касаются друг друга
        if (receiversDistance === firstReceiver.r + secondReceiver.r) {
          const point1 = this.getPointCoords(firstReceiver, secondReceiver, firstReceiver.r, receiversDistance);
          return ({
            twoPoints: false,
            point1,
          });
        }

        // радиусы касаются друг друга и один радиус внутри другого
        if (receiversDistance === Math.abs(firstReceiver.r - secondReceiver.r)) {
          const largerRadiusReceiver = firstReceiver.r > secondReceiver.r ? firstReceiver : secondReceiver
          const smallerRadiusReceiver = firstReceiver.r < secondReceiver.r ? firstReceiver : secondReceiver
          const point1 = this.getPointCoords(
            largerRadiusReceiver,
            smallerRadiusReceiver,
            largerRadiusReceiver.r,
            receiversDistance
          );
          return ({
            twoPoints: false,
            point1,
          });
        }

        // радиусы пересекаются в двух точках
        if (receiversDistance < firstReceiver.r + secondReceiver.r) {
          const receiverToMiddleBetweenPointsDistance =
            (firstReceiver.r ** 2 - secondReceiver.r ** 2 + receiversDistance ** 2) / (2 * receiversDistance);
          const middleBetweenPointsToPointDistance =
            Math.pow(firstReceiver.r ** 2 - receiverToMiddleBetweenPointsDistance ** 2, 1 / 2);
          const middlePointsCoords = this.getPointCoords(
            firstReceiver,
            secondReceiver,
            receiverToMiddleBetweenPointsDistance,
            receiversDistance
          );
          const {point1, point2} = this.getTwoPointsCoords(
            firstReceiver,
            secondReceiver,
            middlePointsCoords,
            middleBetweenPointsToPointDistance,
            receiversDistance
          )
          return ({
            twoPoints: true,
            point1,
            point2,
          });
        }

        // радиусы касаются друг друга и один радиус внутри другого
        if (receiversDistance > Math.abs(firstReceiver.r - secondReceiver.r)) {
          let largerRadiusReceiver = firstReceiver.r > secondReceiver.r ? firstReceiver : secondReceiver
          let smallerRadiusReceiver = firstReceiver.r < secondReceiver.r ? firstReceiver : secondReceiver

          const receiverToMiddleBetweenPointsDistance =
            (largerRadiusReceiver.r ** 2 - smallerRadiusReceiver.r ** 2 + receiversDistance ** 2) / (2 * receiversDistance);
          const middleBetweenPointsToPointDistance =
            Math.pow(largerRadiusReceiver.r ** 2 - receiverToMiddleBetweenPointsDistance ** 2, 1 / 2);

          const middlePointsCoords = this.getPointCoords(
            largerRadiusReceiver,
            smallerRadiusReceiver,
            receiverToMiddleBetweenPointsDistance,
            receiversDistance
          );
          const {point1, point2} = this.getTwoPointsCoords(
            firstReceiver,
            secondReceiver,
            middlePointsCoords,
            middleBetweenPointsToPointDistance,
            receiversDistance
          )
          return ({
            twoPoints: true,
            point1,
            point2,
          });
        }
      },
      getPointCoords(firstReceiver, secondReceiver, receiverToPointDistance, receiversDistance) {
        const point = {};
        const xToDistance = (receiverToPointDistance * Math.abs(firstReceiver.x - secondReceiver.x)) / receiversDistance
        const yToDistance = (receiverToPointDistance * Math.abs(firstReceiver.y - secondReceiver.y)) / receiversDistance

        point.x = firstReceiver.x <= secondReceiver.x ? firstReceiver.x + xToDistance : firstReceiver.x - xToDistance;
        point.y = firstReceiver.y <= secondReceiver.y ? firstReceiver.y + yToDistance : firstReceiver.y - yToDistance;

        return point;
      },
      getTwoPointsCoords(firstReceiver, secondReceiver, middlePointsCoords, middleBetweenPointsToPointDistance, receiversDistance) {
        const xToDistance = (middleBetweenPointsToPointDistance * Math.abs(firstReceiver.x - secondReceiver.x)) / receiversDistance;
        const yToDistance = (middleBetweenPointsToPointDistance * Math.abs(firstReceiver.y - secondReceiver.y)) / receiversDistance;

        const point1 = {x: middlePointsCoords.x + yToDistance}
        const point2 = {x: middlePointsCoords.x - yToDistance}

        if (firstReceiver.y > secondReceiver.y && firstReceiver.x > secondReceiver.x ||
          firstReceiver.y < secondReceiver.y && firstReceiver.x < secondReceiver.x) {
          point1.y = middlePointsCoords.y - xToDistance
          point2.y = middlePointsCoords.y + xToDistance
        } else {
          point1.y = middlePointsCoords.y + xToDistance
          point2.y = middlePointsCoords.y - xToDistance
        }
        return ({point1, point2});
      },
      getPointsDistance(point1, point2) {
        return Math.pow((point1.x - point2.x) ** 2 + (point1.y - point2.y) ** 2, 1 / 2)
      },
      getTriangleCentroid(triangle) {
        return ({
          x: (triangle.firstVertex.x + triangle.secondVertex.x + triangle.thirdVertex.x) / 3,
          y: (triangle.firstVertex.y + triangle.secondVertex.y + triangle.thirdVertex.y) / 3,
        })
      }
    },
  }
</script>

<style>
    #app {
        margin-left: 50px;
        display: flex;
        justify-content: space-between;
        min-width: 1500px;
    }

    .error {
        color: red;
    }

    .upload-file {
        text-align: center;
        width: 1000px;
    }

    .coordinates {
        margin-right: 100px;
    }
</style>
