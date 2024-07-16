<template>
  <div class="canvas-container" ref="canvasContainer">
    <canvas ref="canvas" width="800" height="600"></canvas>
  </div>
</template>

<script>
import data from "@/data.json";

export default {
  name: "ForceDirectedGraph",
  data() {
    return {
      // 初始化节点，设置随机位置和速度，固定位置为null
      nodes: data.nodes.map(node => ({
        ...node,
        x: Math.random() * 800,
        y: Math.random() * 600,
        vx: 0,
        vy: 0,
        fx: null,
        fy: null
      })),
      // 初始化连接
      links: data.links,
      width: 800,
      height: 600,
      draggingNode: null,  // 当前拖动的节点
      transform: {
        x: 0,
        y: 0,
        k: 1
      },
      isPanning: false,  // 是否处于拖动画布状态
      lastMousePos: { x: 0, y: 0 }  // 最后一次鼠标位置
    };
  },
  mounted() {
    this.createForceDirectedGraph();  // 在组件挂载后创建力导向图
  },
  methods: {
    createForceDirectedGraph() {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");
      const alpha = 0.1; // 阻尼系数

      // 为画布添加鼠标事件监听器
      canvas.addEventListener("mousedown", this.onMouseDown);
      canvas.addEventListener("mousemove", this.onMouseMove);
      canvas.addEventListener("mouseup", this.onMouseUp);
      canvas.addEventListener("wheel", this.onMouseWheel);

      // 模拟步骤函数
      const simulationStep = () => {
        this.links.forEach(link => {
          const source = this.nodes.find(n => n.id === link.source);
          const target = this.nodes.find(n => n.id === link.target);
          const dx = target.x - source.x;
          const dy = target.y - source.y;
          const distance = Math.sqrt(dx * dx + dy * dy);
          const force = (distance - 100) / distance * 0.1; // 理想距离是100

          const fx = force * dx;
          const fy = force * dy;

          // 如果节点没有固定位置，更新速度
          if (!source.fx) {
            source.vx += fx;
            source.vy += fy;
          }
          if (!target.fx) {
            target.vx -= fx;
            target.vy -= fy;
          }
        });

        // 更新节点位置和速度
        this.nodes.forEach(node => {
          if (!node.fx) {
            node.vx *= alpha;
            node.vy *= alpha;

            node.x += node.vx;
            node.y += node.vy;

            // 防止节点超出画布边界
            node.x = Math.max(0, Math.min(this.width, node.x));
            node.y = Math.max(0, Math.min(this.height, node.y));
          } else {
            node.x = node.fx;
            node.y = node.fy;
          }
        });

        this.render(ctx);  // 渲染画布

        requestAnimationFrame(simulationStep);  // 请求下一帧动画
      };

      simulationStep();
    },
    render(ctx) {
      ctx.clearRect(0, 0, this.width, this.height);  // 清除画布
      ctx.save();
      ctx.translate(this.transform.x, this.transform.y);  // 应用平移变换
      ctx.scale(this.transform.k, this.transform.k);  // 应用缩放变换

      // 绘制连接
      this.links.forEach(link => {
        const source = this.nodes.find(n => n.id === link.source);
        const target = this.nodes.find(n => n.id === link.target);

        ctx.beginPath();
        ctx.moveTo(source.x, source.y);
        ctx.lineTo(target.x, target.y);
        ctx.strokeStyle = `rgba(0,0,0,${link.value / 10})`;
        ctx.lineWidth = 2;  // 固定边的粗细值
        ctx.stroke();
      });

      // 绘制节点
      this.nodes.forEach(node => {
        ctx.beginPath();
        ctx.arc(node.x, node.y, 5, 0, 2 * Math.PI, false);
        ctx.fillStyle = this.getColor(node.group);  // 根据节点组获取颜色
        ctx.fill();
        ctx.strokeStyle = '#000';
        ctx.stroke();
      });

      ctx.restore();
    },
    getColor(group) {
      const colors = ["#1f77b4", "#ff7f0e", "#2ca02c", "#d62728", "#9467bd", "#8c564b", "#e377c2", "#7f7f7f", "#bcbd22", "#17becf"];
      return colors[group % colors.length];  // 根据组返回颜色
    },
    onMouseDown(event) {
      const mousePos = this.getMousePos(event);
      this.draggingNode = this.getNodeAtPos(mousePos);  // 检查鼠标点击位置是否在节点上

      if (this.draggingNode) {
        // 固定节点位置
        this.draggingNode.fx = this.draggingNode.x;
        this.draggingNode.fy = this.draggingNode.y;
      } else {
        // 开始拖动画布
        this.isPanning = true;
        this.lastMousePos = mousePos;
      }
    },
    onMouseMove(event) {
      const mousePos = this.getMousePos(event);

      if (this.draggingNode) {
        // 更新节点固定位置
        this.draggingNode.fx = mousePos.x;
        this.draggingNode.fy = mousePos.y;
      } else if (this.isPanning) {
        // 更新画布平移
        const dx = mousePos.x - this.lastMousePos.x;
        const dy = mousePos.y - this.lastMousePos.y;

        this.transform.x += dx;
        this.transform.y += dy;

        this.lastMousePos = mousePos;
      }
    },
    onMouseUp() {
      if (this.draggingNode) {
        // 释放节点
        this.draggingNode.fx = null;
        this.draggingNode.fy = null;
        this.draggingNode = null;
      } else if (this.isPanning) {
        this.isPanning = false;  // 停止拖动画布
      }
    },
    onMouseWheel(event) {
      const mousePos = this.getMousePos(event);
      const zoom = event.deltaY > 0 ? 0.9 : 1.1;

      // 更新缩放比例和画布平移
      this.transform.k *= zoom;
      this.transform.x = mousePos.x - (mousePos.x - this.transform.x) * zoom;
      this.transform.y = mousePos.y - (mousePos.y - this.transform.y) * zoom;

      event.preventDefault();
    },
    getMousePos(event) {
      const rect = this.$refs.canvas.getBoundingClientRect();
      return {
        x: (event.clientX - rect.left - this.transform.x) / this.transform.k,
        y: (event.clientY - rect.top - this.transform.y) / this.transform.k
      };
    },
    getNodeAtPos(pos) {
      return this.nodes.find(node => {
        const dx = node.x - pos.x;
        const dy = node.y - pos.y;
        return Math.sqrt(dx * dx + dy * dy) < 5;  // 检查鼠标是否在节点范围内
      });
    }
  }
};
</script>

<style scoped>
.canvas-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100vh;
}
canvas {
  border: 1px solid #ccc;
}
</style>
