FROM docker.io/nvidia/cuda:11.8.0-cudnn8-devel-ubi8

USER 0

### Got below from github: https://github.com/redhat-na-ssa/demo-ai-gitops-catalog/blob/main/components/containers/udi-cuda/ubi8/Dockerfile

# install: common tools
# hadolint ignore=DL3033
RUN yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm && \
    yum install -y bash curl diffutils git git-lfs iproute jq less lsof man nano procps p7zip p7zip-plugins \
                   perl-Digest-SHA net-tools openssh-clients rsync socat sudo time vim wget zip && \
                   yum clean all && \
                   rm -rf /var/cache/yum/*

# install: other bins
RUN yum install -y libGL bash-completion && \
    yum clean all && \
    rm -rf /var/cache/yum/*

# install: oc,tkn
# todo: install cli tools during env init / not in container
ARG OC4_URL="https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz"
ARG TKN_URL="https://mirror.openshift.com/pub/openshift-v4/clients/pipeline/latest/tkn-linux-amd64.tar.gz"

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN curl -sL "${OC4_URL}" | \
      tar -C /usr/local/bin -vzxf- oc kubectl && \
      /usr/local/bin/oc completion bash > /etc/bash_completion.d/oc && \
      echo "source /etc/bash_completion.d/oc" >> /etc/skel/.bashrc && \
    curl -sL "${TKN_URL}" | \
      tar -C /usr/local/bin -vzxf- --no-same-owner tkn && \
      /usr/local/bin/tkn completion bash > /etc/bash_completion.d/tkn && \
      echo "source /etc/bash_completion.d/tkn" >> /etc/skel/.bashrc

###

RUN chmod g+rw /etc/shadow && echo "user ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/should_not_be_done

RUN dnf -y update && \
    dnf -y install python3.9

USER 1001
