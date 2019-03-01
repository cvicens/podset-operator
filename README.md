# New project to test our operator
oc new-project prodset

# Create ccaffold
operator-sdk new podset-operator --type=go --skip-git-init

# Adding custom API
operator-sdk add api --api-version=app.cvicens.com/v1alpha1 --kind=PodSet

# Create CRD in our project
oc create -f deploy/crds/app_v1alpha1_podset_crd.yaml ; oc get crd


# Edit pkg/apis/app/v1apha1/podset_tpes.go

~~~golang
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// PodSetSpec defines the desired state of PodSet
type PodSetSpec struct {
    Replicas int32 `json:"replicas"`
}

// PodSetStatus defines the observed state of PodSet
type PodSetStatus struct {
    PodNames []string `json:"podNames"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// PodSet is the Schema for the podsets API
// +k8s:openapi-gen=true
type PodSet struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   PodSetSpec   `json:"spec,omitempty"`
    Status PodSetStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// PodSetList contains a list of PodSet
type PodSetList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []PodSet `json:"items"`
}

func init() {
    SchemeBuilder.Register(&PodSet{}, &PodSetList{})
}
~~~

# After changing *_types.go files...

~~~shell
operator-sdk generate k8s
~~~

# Add a new Controller to the project that will watch and reconcile the PodSet resource

Next command generates a controller here go/src/github.com/redhat/podset-operator/pkg/controller/podset/podset_controller.go

~~~shell
operator-sdk add controller --api-version=app.cvicens.com/v1alpha1 --kind=PodSet
~~~

# Customize the Operator Logic

~~~golang
package podset

import (
    "context"
    "reflect"

    appv1alpha1 "github.com/redhat/podset-operator/pkg/apis/app/v1alpha1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/manager"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    logf "sigs.k8s.io/controller-runtime/pkg/runtime/log"
    "sigs.k8s.io/controller-runtime/pkg/source"
)

var log = logf.Log.WithName("controller_podset")

/**
* USER ACTION REQUIRED: This is a scaffold file intended for the user to modify with their own Controller
* business logic.  Delete these comments after modifying this file.*
 */

// Add creates a new PodSet Controller and adds it to the Manager. The Manager will set fields on the Controller
// and Start it when the Manager is Started.
func Add(mgr manager.Manager) error {
    return add(mgr, newReconciler(mgr))
}

// newReconciler returns a new reconcile.Reconciler
func newReconciler(mgr manager.Manager) reconcile.Reconciler {
    return &ReconcilePodSet{client: mgr.GetClient(), scheme: mgr.GetScheme()}
}

// add adds a new Controller to mgr with r as the reconcile.Reconciler
func add(mgr manager.Manager, r reconcile.Reconciler) error {
    // Create a new controller
    c, err := controller.New("podset-controller", mgr, controller.Options{Reconciler: r})
    if err != nil {
        return err
    }

    // Watch for changes to primary resource PodSet
    err = c.Watch(&source.Kind{Type: &appv1alpha1.PodSet{}}, &handler.EnqueueRequestForObject{})
    if err != nil {
        return err
    }

    // TODO(user): Modify this to be the types you create that are owned by the primary resource
    // Watch for changes to secondary resource Pods and requeue the owner PodSet
    err = c.Watch(&source.Kind{Type: &corev1.Pod{}}, &handler.EnqueueRequestForOwner{
        IsController: true,
        OwnerType:    &appv1alpha1.PodSet{},
    })
    if err != nil {
        return err
    }

    return nil
}

var _ reconcile.Reconciler = &ReconcilePodSet{}

// ReconcilePodSet reconciles a PodSet object
type ReconcilePodSet struct {
    // This client, initialized using mgr.Client() above, is a split client
    // that reads objects from the cache and writes to the apiserver
    client client.Client
    scheme *runtime.Scheme
}

// Reconcile reads that state of the cluster for a PodSet object and makes changes based on the state read
// and what is in the PodSet.Spec
// TODO(user): Modify this Reconcile function to implement your Controller logic.  This example creates
// a Pod as an example
// Note:
// The Controller will requeue the Request to be processed again if the returned error is non-nil or
// Result.Requeue is true, otherwise upon completion it will remove the work from the queue.
func (r *ReconcilePodSet) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
    reqLogger.Info("Reconciling PodSet")

    // Fetch the PodSet instance
    podSet := &appv1alpha1.PodSet{}
    err := r.client.Get(context.TODO(), request.NamespacedName, podSet)
    if err != nil {
        if errors.IsNotFound(err) {
            // Request object not found, could have been deleted after reconcile request.
            // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
            // Return and don't requeue
            return reconcile.Result{}, nil
        }
        // Error reading the object - requeue the request.
        return reconcile.Result{}, err
    }

    // List all pods owned by this PodSet instance
    podList := &corev1.PodList{}
    lbs := map[string]string{
        "app":     podSet.Name,
        "version": "v0.1",
    }
    labelSelector := labels.SelectorFromSet(lbs)
    listOps := &client.ListOptions{Namespace: podSet.Namespace, LabelSelector: labelSelector}
    if err = r.client.List(context.TODO(), listOps, podList); err != nil {
        return reconcile.Result{}, err
    }

    // Count the pods that are pending or running as available
    var available []corev1.Pod
    for _, pod := range podList.Items {
        if pod.ObjectMeta.DeletionTimestamp != nil {
            continue
        }
        if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
            available = append(available, pod)
        }
    }
    numAvailable := int32(len(available))
    availableNames := []string{}
    for _, pod := range available {
        availableNames = append(availableNames, pod.ObjectMeta.Name)
    }

    // Update the status if necessary
    status := appv1alpha1.PodSetStatus{
        PodNames: availableNames,
    }
    if !reflect.DeepEqual(podSet.Status, status) {
        podSet.Status = status
        err = r.client.Update(context.TODO(), podSet)
        if err != nil {
            reqLogger.Error(err, "Failed to update PodSet status")
            return reconcile.Result{}, err
        }
    }

    if numAvailable > podSet.Spec.Replicas {
        reqLogger.Info("Scaling down pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
        diff := numAvailable - podSet.Spec.Replicas
        dpods := available[:diff]
        for _, dpod := range dpods {
            err = r.client.Delete(context.TODO(), &dpod)
            if err != nil {
                reqLogger.Error(err, "Failed to delete pod", "pod.name", dpod.Name)
                return reconcile.Result{}, err
            }
        }
        return reconcile.Result{Requeue: true}, nil
    }

    if numAvailable < podSet.Spec.Replicas {
        reqLogger.Info("Scaling up pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
        // Define a new Pod object
        pod := newPodForCR(podSet)
        // Set PodSet instance as the owner and controller
        if err := controllerutil.SetControllerReference(podSet, pod, r.scheme); err != nil {
            return reconcile.Result{}, err
        }
        err = r.client.Create(context.TODO(), pod)
        if err != nil {
            reqLogger.Error(err, "Failed to create pod", "pod.name", pod.Name)
            return reconcile.Result{}, err
        }
        return reconcile.Result{Requeue: true}, nil
    }

    return reconcile.Result{}, nil
}

// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
    labels := map[string]string{
        "app":     cr.Name,
        "version": "v0.1",
    }
    return &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            GenerateName: cr.Name + "-pod",
            Namespace:    cr.Namespace,
            Labels:       labels,
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:    "busybox",
                    Image:   "busybox",
                    Command: []string{"sleep", "3600"},
                },
            },
        },
    }
}
~~~


# Running the Operator Locally (Inside the Cluster)

~~~shell
operator-sdk up local --namespace podset
~~~

# Creating the Custom Resource

Edit deploy/crds/app_v1alpha1_podset_cr.yaml

~~~yaml
apiVersion: app.cvicens.com/v1alpha1
kind: PodSet
metadata:
  name: example-podset
spec:
  # Old property example
  # size: 3
  # New property
  replicas: 3
~~~

Create the new resource...

~~~shell
$ oc create -f deploy/crds/app_v1alpha1_podset_cr.yaml
$ oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
example-podset-podk5mdc   1/1       Running   0          1m
example-podset-podkmr4b   1/1       Running   0          1m
example-podset-podmd7xl   1/1       Running   0          1m
~~~

~~~shell
$ oc get podset example-podset -o yaml
apiVersion: app.cvicens.com/v1alpha1
kind: PodSet
metadata:
  creationTimestamp: 2019-03-01T07:29:42Z
  generation: 1
  name: example-podset
  namespace: podset
  resourceVersion: "2176002"
  selfLink: /apis/app.cvicens.com/v1alpha1/namespaces/podset/podsets/example-podset
  uid: c7fc54d3-3bf3-11e9-a788-0af43ed02106
spec:
  replicas: 3
~~~