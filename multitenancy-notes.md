# Use the keycloak terminal to lauch keycloak
```
docker run -d --name=keycloak -p 8080:8080 -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin quay.io/keycloak/keycloak:16.1.1
```

Wait few seconds that keycloak finish starting.
You can control that by searching the sentence `Admin console listening on http://127.0.0.1:9990` in the logs.

```
docker logs keycloak -f
```

Use Ctrl-C to get out of the logs.

Once keycloak started let's configure keycloak to accept non https protocol, securing the OIDC provider is not the objective
of this track and we take here the shortest path.

```
docker exec -it keycloak bash
```

Change the configuration of the master realm.
```
cd /opt/jboss/keycloak/bin/
./kcadm.sh config credentials --server http://localhost:8080/auth --realm master --user admin
```
Password is admin.

Now set up the configuration
```
./kcadm.sh update realms/master -s sslRequired=NONE
exit
```

You can now open a browser tab to the keycloak url
```
echo http://keycloak.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:8080/
```

and click on the administraion console link.

The login and password were setup in the docker command it's admin/admin.

# Create the my-company realm

We create a new realm call "my-company"

![New realm my-company](../assets/add-realm.png)

Make sure this realm has also SSL not required

![disable SSL for realm my-company](../assets/add-realm-disable-ssl.png)

## Create a k10admin group and a user in this group

Create an admin group k10admin (we'll add other group later)

![Create k10admin group](../assets/add-admin-group.png)

Create the user michael belonging to the k10admingroup. Make sure it has his email verified.

![Create michael](../assets/add-user.png)

Setup the password as michael (same as username for simplicity but feel free to use something else).
Turn off temporary.

![Setup password](../assets/add-user-password.png)

Make sure it has his email verified

![Check email verified](../assets/add-user-email-verified.png)

## Create a kasten client in the realm

Create a kasten client in the realm my-company.
Make sure the client is Enabled.

![Create a kasten client](../assets/add-client.png)

Make sure this client is confidential and accept implicit flow (implicit flow is needed for using oidcdebugger)

![kasten client, confidential and accept implicit flow](../assets/add-client-confidential-implicit.png)

On the allowed redirect add
- https://oidcdebugger.com/debug
- the url of the k10 oidc redirect endpoint (even if kasten has not been created we know what will be the URL)
```
echo http://k8svm.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:32000/k10/auth-svc/v0/oidc/redirect
```

![kasten client, redirects](../assets/add-client-redirects.png)

To add the groups in the JWT token add the group mapper,
- Disable Full group path.
- For name choose what you want (ie group-mapper)
- For Token Claim Name choose `groups`

![kasten client, add the group mapper](../assets/add-client-group-mapper.png)


# Let see what will be the JWT token returned by keycloak

In a new browser tab go to https://oidcdebugger.com/  use

```
curl http://keycloak.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:8080/auth/realms/my-company/.well-known/openid-configuration | jq '.authorization_endpoint'
```

to fill up the field form

![Fill up the form](../assets/oidc-debugger.png)

Then click send and connect as michael/michael

You should be redirected to oidc-debugger with the content of the JWT Token

```
{
   "exp": 1644490827,
   "iat": 1644490527,
   "auth_time": 1644490072,
   "jti": "1857a3b2-ada4-4b21-8449-c058e88a3601",
   "iss": "http://keycloak.5vubxfv1zp9p.instruqt.io:8080/auth/realms/my-company",
   "aud": "kasten",
   "sub": "aa30b2a9-cd13-42d0-baec-449434921d33",
   "typ": "ID",
   "azp": "kasten",
   "nonce": "ornkyd457rq",
   "session_state": "9d014d1d-170f-48f1-907f-0824339bc04d",
   "c_hash": "NhhY4mDF9yaZjl9qu7Gaxw",
   "acr": "0",
   "sid": "9d014d1d-170f-48f1-907f-0824339bc04d",
   "email_verified": true,
   "preferred_username": "michael",
   "groups": [
      "k10admin"
   ]
}
```

Congratulation your keycloak configuration is successful.

# More user and groups

Now in keycloak in the my-company realms create 3 groups dev, test and prod

And those users :

*  dev-user1/dev6v3p9gy, dev-user2/dev6v3p9gy, dev-user3/dev6v3p9gy belonging to the dev group
*  test-user1/test6v3p9gy, test-user2/test6v3p9gy, test-user3/test6v3p9gy belonging to the test group
*  prod-user1/prod6v3p9gy, prod-user2/prod6v3p9gy, prod-user3/prod6v3p9gy belonging to the prod group
*  devops-user1/devops6v3p9gy belonging to the dev, test and prod group

We'll use them latter when testing with RBAC.

In this step, we will actually install K10 with those oidc options.

But as it becomes a complex install instead of using `--set option`
we're going to use a values.yaml file.

```
cat <<EOF > values.yaml
auth:
  oidcAuth:
    enabled: true
    clientID: kasten
    clientSecret: <kasten-secret>
    providerURL: http://keycloak.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:8080/auth/realms/my-company
    redirectURL: http://k8svm.$INSTRUQT_PARTICIPANT_ID.instruqt.io:32000/
    scopes: "openid email profile"
    usernameClaim: preferred_username
    groupClaim: groups
    usernamePrefix: "-"
EOF

```
The options you can read are consistent with the OIDC provider we've setup
and the content of the JWT token we checked in the oidcdebugger.

Those options are described [here](https://docs.kasten.io/latest/access/authentication.html#openid-connect-authentication).


Notice particulary this option about the client secret which is used when working with authorisation code flow and confidential client:

```
    clientSecret: <kasten-secret>
```

You can find the secret in the credential tab of the kasten client in Keycloack.

![kasten client credential](../assets/kasten-client-secret.png)

Replace the value of the secret in the values.yaml file.

Let's install now :
```console
helm repo add kasten https://charts.kasten.io/
helm repo update
kubectl create ns kasten-io
kubectl create -f /root/license-secret.yaml

helm install k10 kasten/k10 --namespace kasten-io -f values.yaml
```


To ensure that Kasten K10 is running, check the pod status to make sure they are all in the `Running` state:
```console
watch -n 2 "kubectl -n kasten-io get pods"
```

Once all pods have a Running status, hit `CTRL + C` to exit `watch`.

# Configure the Local Storage System

Once K10 is running, use the following commands to configure the local storage system.
```console
kubectl annotate volumesnapshotclass csi-hostpath-snapclass k10.kasten.io/is-snapshot-class=true
```

# Expose the K10 dashboard

While not recommended for production environments, let's set up access to the K10 dashboard by creating a NodePort. Let's first create the configuration file for this:

```console
cat > k10-nodeport-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: gateway-nodeport
  namespace: kasten-io
spec:
  selector:
    service: gateway
  ports:
  - name: http
    port: 8000
    nodePort: 32000
  type: NodePort
EOF
```

Now, let's create the actual NodePort Service

```console
kubectl apply -f k10-nodeport-svc.yaml
```
# View the K10 Dashboard and check your identity

Once completed, you should be able to view the K10 dashboard in the other tab on the left.

You'll be redirected to the keycloak login window if you have not been authentified.

Identify first as michael/michael and check you have proper username

![Connected as Michael](../assets/session-michael-username.png)

And check the groups you belong to :
groups Settings > Support > View Current User Details

![Michael'group](../assets/session-settings.png)

![Michael'group](../assets/session-michael-group.png)

# Logout

For different reasons that are beyond the scope of this track we decide to
not trigger a logout at the oidc provider level when you logout as michael.

If you simply logout from Kasten you will be automatically logged in again
because you are still logged in at the oidc provider.

You need to explicitly open the OIDC provider page

```
echo http://keycloak.${INSTRUQT_PARTICIPANT_ID}.instruqt.io:8080/auth/realms/my-company/account/#/
```

![Keycloak signout](../assets/session-michael-signout.png)

Now you can singn out on kasten and sign in again with another user

# Login as dev-user1, dev-user2 and devops-user1 and check groups

Now login as dev-user1 or dev-user2 and check your group, you should see in both case one group : dev.

![dev-user1 groups](../assets/session-dev-user1-group.png)

Log out and login as devops-user1 you should see 3 groups dev, test and prod

![devops-user1 groups](../assets/session-devops-user1-group.png)

# Conclusion

Our connection between the OIDC provider and Kasten is working really well and leave
us a great flexibility in managing group membership

But for the moment whatever the user there is nothing we can do ...

Tiles show 0 applications and 0 policies. How that happens ?

![devops-user1 no apps](../assets/session-dev-user1-0-apps.png)

It's here that RBAC will come into play, and that's for the next challenges.

# Identify built-in cluster role

When you install kasten it comes with built in Kasten ClusterRole.
You can attach a ClusterRole
*  in a ClusterRoleBinding (which is not a namespaced object) and in this case the role is applied for the user in all the namespaces
*  in a RoleBinding (which is a namespaced object) and in this case the role is applied for the user only in this namespace
List all the built in ClusterRole brought during Kasten installation

```console
kubectl get clusterrole |grep k10
```

List all the built in role in the kasten-io namespace brought during Kasten installation
```console
kubectl get role -n kasten-io
```

Those ClusterRole/Role are described in the [kasten RBAC documentation](https://docs.kasten.io/latest/access/rbac.html).

# Give michael the k10-admin role on all the namespaces

michael belongs to the group k10admin, let's see if the group `k10admin` can create a backupaction in any namespace ?

```console
kubectl auth can-i create backupactions.actions.kio.kasten.io --as=michael --as-group=k10admin --all-namespaces
```

The answer should be no...

You must understand that each time the dashboard backend is acting on the API it's doing so the same
way we jus did it in this API call :

```
--as=michael --as-group=k10admin
```

But even if it's possible we don't want to set up a rolebinding at the user level because for big
organisations with many teams that would be very difficult to maintain. It's why we set it up at the
group level.

Let's apply a clusterrolebinding then

```console
kubectl create clusterrolebinding k10-admin-group-k10admin --clusterrole=k10-admin --group=k10admin
```

And now I can ask again my previous question

```
kubectl auth can-i create backupactions.actions.kio.kasten.io --as=michael --as-group=k10admin --all-namespaces
```

This time the answer is yes ! And you'll see a big change in the kasten dashboard as well. All the namespaces
are now available and if you check the  Settings > Support > Cluster information > View Current User Details
you can see that you can now perform all operations.

But you notice also that you constantly get this popup saying "K10 services are reporting errors". Actually
There is no errors but because you are k10-admin on all namespaces - kasten-io included, the dashboard impersonate
`k10admin` group to check the health of the kasten-io namespace.

But the k10-admin ClusterRole is only made for manipulating Kasten API not deployment or secret. So for this special
case we need to bind the local role kasten-ns-admin in the kasten-io namespace to the group k10admin :

```console
 kubectl create rolebinding k10-ns-admin-group-k10admin --role=k10-ns-admin --group=k10admin --namespace=kasten-io
```

Now with a user belonging to the k10admin group you have a working k10-admin among your users.

Let's check that michael is really able to use Kasten, create a policy for the `default` namespace with all the default option and run it once.

For the need of this tutorial let's create 3 namespaces

```
kubectl create ns development
kubectl create ns qualification
kubectl create ns production
```

Connect as dev-user1 for the moment you can't see any namespaces on the dasboard, specifically not on the development namespace.

 ```
 kubectl auth can-i create backupactions.actions.kio.kasten.io --as-group=dev --as=dev-user1 -n development
 ```

 The answer should be no.

You must understand that each time the dashboard backend is acting on the API it's doing so the same
way we jus did it in this API call :

```
--as=dev-user1 --as-group=dev
```

 We want to limit access for `dev-user1` to only the `development` namespace where he'll be able to perform backup and restore
 of this namespace.

 Check the content of the Role kasten-basic

 ```
 kubectl get clusterrole k10-basic -o yaml
 ```

 You can see the diffent action a basic-user can do particulary create BackupAction or RestoreAction.

 Let's dev-user1 be a basic-user on this namespace by acting on his group.

 ```
 kubectl create rolebinding k10-basic-group-dev --clusterrole=k10-basic --group=dev --namespace=development
 ```

 Now let check that alice can create backup action
 ```
 kubectl auth can-i create backupactions.actions.kio.kasten.io --as-group=dev --as=dev-user1 --namespace=development
 ```

 The answer should be yes this time.

 But if we ask the same question for qualification
```
 kubectl auth can-i create backupactions.actions.kio.kasten.io --as-group=dev --as=dev-user1 --namespace=qualification
 ```

The answer should be no, it's a namespace binding contrary to a cluster binding as we did for michael.

Go in the dashboard. You should see the development namespace and only the development namespace.

Try to perfom a backup by going to the application tile and click the "snapshot" button.

WARNING: Non k10-admin user can create policy on their own but won't have the right to run them manually,
only the scheduler will be able to run them.

 # challenge RBAC

 Now it's your turn, make the necessary operation for having test group able to perform
 k10-basic operations on qualification namespaces and prod group able to perform K10-basic
 operations on the production namespace.

 Check by connecting with
 - test-user1 you can only perform k10-basic operations on the qualification namespace
 - prod-user1 you can only perform k10-basic operations on the production namespace
 - devops-user1 you can perform k10-basic operation on the 3 namespaces : development, qualification and production
