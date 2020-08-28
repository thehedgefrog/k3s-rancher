kubernetes.io/ingress.class=traefik
ingress.kubernetes.io/auth-type=forward
ingress.kubernetes.io/auth-url=http://login.mydomain.com/api/verify?rd=https://login.mydomain.com/
traefik.ingress.kubernetes.io/redirect-entry-point=https
traefik.ingress.kubernetes.io/redirect-permanent=true