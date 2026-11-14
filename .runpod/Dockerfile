# Hub builds use the release image after its CPU installation/schema checks pass.
FROM ghcr.io/acdevmedia/erosh3-comfyui:h3-beta7-fix1
COPY .runpod/handler.py /opt/video/handler.py
ENV PORT=8000 PORT_HEALTH=8001
EXPOSE 8188 8000 8001 3923
ENTRYPOINT ["/usr/bin/tini", "-g", "--", "python3", "/opt/video/handler.py", "bash", "/opt/video/scripts/start.sh"]
