name: 5节点全主高可用集群(最终优化版)
run-name: HA-Cluster-全自动-永不宕机

on:
  workflow_dispatch:
  repository_dispatch:

permissions:
  contents: write
  actions: write

# 👇 你只需要修改这里 4 个值 👇
env:
  TAILSCALE_KEY: "tskey-auth-你自己的-Tailscale密钥"
  SSH_USER: "root"
  SSH_PASS: "你的统一密码"
  OWNER_REPO: "你的GitHub用户名/仓库名"

jobs:
  # ========== 节点1：初始化集群主节点 ==========
  node1:
    runs-on: ubuntu-latest
    outputs:
      manager_ip: ${{ steps.ts_ip.outputs.ip }}
    steps:
      - name: 安装系统依赖
        run: |
          apt update -y
          apt install -y curl wget openssh-server glusterfs-server sshpass
          systemctl enable --now ssh glusterd
          echo "${SSH_USER}:${SSH_PASS}" | chpasswd
          sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
          sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart ssh

      - name: 安装并加入Tailscale内网
        run: |
          curl -fsSL https://tailscale.com/install.sh | sh
          tailscale up --authkey=${TAILSCALE_KEY} --accept-dns --accept-routes --force

      - name: 获取Tailscale内网IP
        id: ts_ip
        run: echo "ip=$(tailscale ip -4)" >> "$GITHUB_OUTPUT"

      - name: 安装Docker并初始化Swarm全主集群
        run: |
          curl -fsSL https://get.docker.com | sh
          usermod -aG docker ${SSH_USER}
          docker swarm init --advertise-addr ${{ steps.ts_ip.outputs.ip }}
          mkdir -p /data/share && chmod 777 /data/share

      - name: 自动续跑 + 永久保活
        run: |
          (sleep 21000 && curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/${OWNER_REPO}/actions/workflows/ha-cluster.yml/dispatches \
            -d '{"ref":"main"}') &
          while true; do echo "✅ node1 主节点+干活中 | $(date)"; sleep 60; done

  # ========== 节点2：全主节点 ==========
  node2:
    runs-on: ubuntu-latest
    needs: node1
    steps:
      - name: 初始化环境
        run: |
          apt update -y
          apt install -y curl wget openssh-server glusterfs-server sshpass
          systemctl enable --now ssh glusterd
          echo "${SSH_USER}:${SSH_PASS}" | chpasswd
          sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
          sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart ssh
          curl -fsSL https://tailscale.com/install.sh | sh
          curl -fsSL https://get.docker.com | sh
          usermod -aG docker ${SSH_USER}
      - name: 加入内网
        run: tailscale up --authkey=${TAILSCALE_KEY} --accept-dns --accept-routes --force
      - name: 以主节点身份加入集群
        run: |
          sshpass -p ${SSH_PASS} ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${SSH_USER}@${{ needs.node1.outputs.manager_ip }} \
          "docker swarm join-token manager" | grep -oP 'docker swarm join .*' > join.sh
          chmod +x join.sh && ./join.sh
          mkdir -p /data/share && chmod 777 /data/share
      - name: 自动续跑保活
        run: |
          (sleep 21000 && curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${OWNER_REPO}/actions/workflows/ha-cluster.yml/dispatches \
            -d '{"ref":"main"}') &
          while true; do echo "✅ node2 主节点+干活中"; sleep 60; done

  # ========== 节点3：全主节点 ==========
  node3:
    runs-on: ubuntu-latest
    needs: node1
    steps:
      - name: 初始化环境
        run: |
          apt update -y
          apt install -y curl wget openssh-server glusterfs-server sshpass
          systemctl enable --now ssh glusterd
          echo "${SSH_USER}:${SSH_PASS}" | chpasswd
          sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
          sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart ssh
          curl -fsSL https://tailscale.com/install.sh | sh
          curl -fsSL https://get.docker.com | sh
          usermod -aG docker ${SSH_USER}
      - name: 加入内网
        run: tailscale up --authkey=${TAILSCALE_KEY} --accept-dns --accept-routes --force
      - name: 以主节点身份加入集群
        run: |
          sshpass -p ${SSH_PASS} ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${SSH_USER}@${{ needs.node1.outputs.manager_ip }} \
          "docker swarm join-token manager" | grep -oP 'docker swarm join .*' > join.sh
          chmod +x join.sh && ./join.sh
          mkdir -p /data/share && chmod 777 /data/share
      - name: 自动续跑保活
        run: |
          (sleep 21000 && curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${OWNER_REPO}/actions/workflows/ha-cluster.yml/dispatches \
            -d '{"ref":"main"}') &
          while true; do echo "✅ node3 主节点+干活中"; sleep 60; done

  # ========== 节点4：全主节点 ==========
  node4:
    runs-on: ubuntu-latest
    needs: node1
    steps:
      - name: 初始化环境
        run: |
          apt update -y
          apt install -y curl wget openssh-server glusterfs-server sshpass
          systemctl enable --now ssh glusterd
          echo "${SSH_USER}:${SSH_PASS}" | chpasswd
          sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
          sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart ssh
          curl -fsSL https://tailscale.com/install.sh | sh
          curl -fsSL https://get.docker.com | sh
          usermod -aG docker ${SSH_USER}
      - name: 加入内网
        run: tailscale up --authkey=${TAILSCALE_KEY} --accept-dns --accept-routes --force
      - name: 以主节点身份加入集群
        run: |
          sshpass -p ${SSH_PASS} ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${SSH_USER}@${{ needs.node1.outputs.manager_ip }} \
          "docker swarm join-token manager" | grep -oP 'docker swarm join .*' > join.sh
          chmod +x join.sh && ./join.sh
          mkdir -p /data/share && chmod 777 /data/share
      - name: 自动续跑保活
        run: |
          (sleep 21000 && curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${OWNER_REPO}/actions/workflows/ha-cluster.yml/dispatches \
            -d '{"ref":"main"}') &
          while true; do echo "✅ node4 主节点+干活中"; sleep 60; done

  # ========== 节点5：全主节点 ==========
  node5:
    runs-on: ubuntu-latest
    needs: node1
    steps:
      - name: 初始化环境
        run: |
          apt update -y
          apt install -y curl wget openssh-server glusterfs-server sshpass
          systemctl enable --now ssh glusterd
          echo "${SSH_USER}:${SSH_PASS}" | chpasswd
          sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
          sed -i 's/^PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart ssh
          curl -fsSL https://tailscale.com/install.sh | sh
          curl -fsSL https://get.docker.com | sh
          usermod -aG docker ${SSH_USER}
      - name: 加入内网
        run: tailscale up --authkey=${TAILSCALE_KEY} --accept-dns --accept-routes --force
      - name: 以主节点身份加入集群
        run: |
          sshpass -p ${SSH_PASS} ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${SSH_USER}@${{ needs.node1.outputs.manager_ip }} \
          "docker swarm join-token manager" | grep -oP 'docker swarm join .*' > join.sh
          chmod +x join.sh && ./join.sh
          mkdir -p /data/share && chmod 777 /data/share
      - name: 自动续跑保活
        run: |
          (sleep 21000 && curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${OWNER_REPO}/actions/workflows/ha-cluster.yml/dispatches \
            -d '{"ref":"main"}') &
          while true; do echo "✅ node5 主节点+干活中"; sleep 60; done
