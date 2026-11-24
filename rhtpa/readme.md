# Set ENVIRONMENT VARIABLES

```
# like apps.cluster-gzfxp.dyn.redhatworkshops.io
export CLUSTER_DOMAIN=""
```

# Install Red Hat Trusted Profile Analyzer Operator

```
envsubst < operator/1-namespace.yaml | oc apply -f -
envsubst < operator/2-operator-group.yaml | oc apply -f -
envsubst < operator/3-subscription.yaml | oc apply -f -
```

# Install TPA Database (pre-req) then TPA
```
envsubst < instance/0-tssc-tpa-database.yaml | oc apply -f -

# update with the tssc-tpa-cli client secret (from tssc-tpa realm)
envsubst < instance/1-tssc-tpa-oidc-cli-credentials.yaml | oc apply -f -
envsubst < instance/2-tssc-tpa-storage.yaml | oc apply -f -

# Set internal DNS
# If you hit any issue using self-signed certificates, try to use S3 Nooba External DNS
# noobaa status -n openshift-storage |& grep 'Address\|DNS'
export TPA_BUCKET_HOST=$(oc get configmap tssc-tpa-bucket -n trusted-profile-analyzer -o jsonpath='{.data.BUCKET_HOST'})
export TPA_BUCKET_NAME=$(oc get configmap tssc-tpa-bucket -n trusted-profile-analyzer -o jsonpath='{.data.BUCKET_NAME'})
export TPA_BUCKET_PORT=$(oc get configmap tssc-tpa-bucket -n trusted-profile-analyzer -o jsonpath='{.data.BUCKET_PORT'})
export TPA_BUCKET_ACCESS_KEY=$(oc get secret tssc-tpa-bucket -n trusted-profile-analyzer -o go-template='{{index .data "AWS_ACCESS_KEY_ID"}}')
export TPA_BUCKET_SECRET=$(oc get secret tssc-tpa-bucket -n trusted-profile-analyzer -o go-template='{{index .data "AWS_SECRET_ACCESS_KEY"}}')

envsubst < instance/3-tssc-tpa-storage-credentials.yaml | oc apply -f -

envsubst < instance/4-tssc-tpa-database-user-credentials.yaml | oc apply -f -
envsubst < instance/5-tssc-tpa-database-admin-credentials.yaml | oc apply -f -
envsubst < instance/7-tssc-tpa-instance.yaml | oc apply -f -
```

obs: update the files instance/1-tssc-tpa-oidc-cli-credentials.yaml and instance/3-tssc-tpa-storage-credentials.yaml before applying
