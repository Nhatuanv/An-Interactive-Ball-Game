
Readme

To enforce that only one object responds to a tap at any given time, you need a way to “lock out” new interactions while one is already running. In your setup you already have an interaction queue in the InteractionQueueManager and a per-object interactable flag in TapAnimationController2. The missing piece is a global check so that when one object is processing its interaction sequence, taps on any other object are ignored.

One effective solution is to expose a global (or manager-based) property that indicates an active interaction. Then, in each object’s OnTapped handler, you can check that property before enqueuing a new interaction.

Below are the changes you can apply:

1. Update the InteractionQueueManager

Add a public property (or a static flag) that tells you whether an interaction is active. For example, add an IsInteractionActive property that returns true if the manager is processing or if there are pending interactions.

// InteractionQueueManager.cs
using UnityEngine;
using System.Collections.Generic;
using System.Collections;

public class InteractionQueueManager : MonoBehaviour
{
    public static InteractionQueueManager Instance { get; private set; }

    private Queue<TapAnimationController2> _interactionQueue = new Queue<TapAnimationController2>();
    private bool _isProcessing = false;

    // Global property to check if any interaction is active
    public bool IsInteractionActive
    {
        get { return _isProcessing || _interactionQueue.Count > 0; }
    }

    void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    public void EnqueueInteraction(TapAnimationController2 controller)
    {
        // Only enqueue if no interaction is active
        if (IsInteractionActive)
            return;
            
        _interactionQueue.Enqueue(controller);
        controller.SetInteractionState(false);

        if (!_isProcessing)
            StartCoroutine(ProcessQueue());
    }

    private IEnumerator ProcessQueue()
    {
        _isProcessing = true;

        while (_interactionQueue.Count > 0)
        {
            var current = _interactionQueue.Dequeue();
            current.SetInteractionState(true);
            yield return current.ExecuteInteractionSequence();
            current.SetInteractionState(false);
        }

        _isProcessing = false;
    }
}

(See  ￼ for your original InteractionQueueManager.cs)

In this updated version, the EnqueueInteraction method first checks if an interaction is already active (via the IsInteractionActive property). If so, it simply ignores the new tap instead of adding it to the queue.

2. Update TapAnimationController2

Now, in your TapAnimationController2 script, modify the OnTapped method to check the manager’s global flag. This prevents an object from being enqueued if another interaction is already in progress.

// TapAnimationController2.cs
using UnityEngine;
using TouchScript.Gestures;
using System.Collections;

[RequireComponent(typeof(TapGesture), typeof(Collider), typeof(Animator))]
public class TapAnimationController2 : MonoBehaviour
{
    [Header("Animation Settings")]
    [SerializeField] private string triggerParam = "popTrigger";

    [Header("3D Text Settings")]
    [SerializeField] private Animator textAnimator;
    [SerializeField] private string textTriggerParam = "showTextTrigger";
    [SerializeField] private float textAnimationDelay = 0.1f;

    [Header("Movement Settings")]
    [SerializeField] private LayerMask touchableLayers;
    [SerializeField] private float rayDistance = 100f;
    [SerializeField] private float smoothFactor = 0.03f;
    [SerializeField] private float movementSpeed = 3f;

    [Header("Floating Control")]
    [SerializeField] private FloatingMovement floatingMovement;

    private Animator _animator;
    private TapGesture _tapGesture;
    private Collider _collider;
    private Vector3 _targetPosition;
    private Vector3 _velocity = Vector3.zero;
    private bool _isMoving = false;
    private bool _isInteractable = true;

    void Awake()
    {
        _animator = GetComponent<Animator>();
        _tapGesture = GetComponent<TapGesture>();
        _collider = GetComponent<Collider>();

        if (!floatingMovement)
            floatingMovement = GetComponent<FloatingMovement>();
    }

    void OnEnable() => _tapGesture.Tapped += OnTapped;
    void OnDisable() => _tapGesture.Tapped -= OnTapped;

    void Update()
    {
        if (_isMoving)
        {
            transform.position = Vector3.SmoothDamp(
                transform.position,
                _targetPosition,
                ref _velocity,
                smoothFactor,
                Mathf.Infinity,
                Time.deltaTime * movementSpeed
            );

            if (Vector3.Distance(transform.position, _targetPosition) < 0.1f)
            {
                _isMoving = false;
                if (floatingMovement != null)
                {
                    floatingMovement.enabled = true;
                    floatingMovement.ResetFloating();
                }
            }
        }
    }

    void OnTapped(object sender, System.EventArgs e)
    {
        // Check both local interactable state and global interaction state.
        if (!_isInteractable || InteractionQueueManager.Instance.IsInteractionActive)
            return;

        if (!Physics.Raycast(
            Camera.main.ScreenPointToRay(_tapGesture.ScreenPosition),
            out RaycastHit hit,
            rayDistance,
            touchableLayers
        ))
            return;

        if (hit.transform != transform)
            return;

        InteractionQueueManager.Instance.EnqueueInteraction(this);
    }

    public void SetInteractionState(bool isActive)
    {
        // When isActive is true, the interaction is in progress so disable this object’s tap collider.
        _isInteractable = !isActive;
        _collider.enabled = !isActive;
    }

    public IEnumerator ExecuteInteractionSequence()
    {
        // Stop floating movement.
        if (floatingMovement != null)
        {
            floatingMovement.ForceStop();
            floatingMovement.enabled = false;
        }

        // Start main animation.
        _animator.SetTrigger(triggerParam);

        // Hide text if the central controller exists.
        if (TextController.Instance != null)
            TextController.Instance.HideIntroText();

        // Move the object to the center.
        MoveToCenter();
        yield return new WaitWhile(() => _isMoving);

        // Display the text.
        yield return ShowText();

        // Re-enable interactions.
        SetInteractionState(false);
    }

    IEnumerator ShowText()
    {
        yield return new WaitForSeconds(textAnimationDelay);
        textAnimator.transform.position = transform.position + Vector3.up * 1.5f;
        textAnimator.SetTrigger(textTriggerParam);
    }

    void MoveToCenter()
    {
        Vector3 screenCenter = new Vector3(0.5f, 0.4f, Camera.main.nearClipPlane + 14f);
        _targetPosition = Camera.main.ViewportToWorldPoint(screenCenter);
        _velocity = Vector3.zero;
        _isMoving = true;
    }
}

(See  ￼ for your original TapAnimationController2.cs)

By checking the global IsInteractionActive property before enqueuing, you ensure that while one object is executing its sequence, taps on any other object are ignored.

3. How It Works
	•	Global Lock via the Manager:
The InteractionQueueManager now exposes an IsInteractionActive property that returns true when either an interaction is being processed or there’s a pending item in the queue.
	•	Preventing New Taps:
In each TapAnimationController2 instance, the OnTapped method first verifies that the object is locally interactable and that no other interaction is active. This guarantees that if one object is tapped, any additional taps on the remaining objects will be ignored until the active sequence finishes.
	•	Sequential Execution:
Once an object is enqueued, its interaction sequence runs. Only after that sequence (including its animations and movement) is complete will the global flag be cleared, allowing a new tap to be processed.

With these modifications, your 7 floating objects will now animate one by one. When one object is tapped, the others will not respond until the current object’s animation sequence is completely finished.