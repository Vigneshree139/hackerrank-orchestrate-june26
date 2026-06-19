import pandas as pd
import os

# Load datasets
claims = pd.read_csv("dataset/claims.csv")
user_history = pd.read_csv("dataset/user_history.csv")
evidence_req = pd.read_csv("dataset/evidence_requirements.csv")

def extract_claim_details(user_claim_text):
    """
    NLP parsing of claim transcript.
    Example: "My laptop screen is cracked"
    -> issue_type = "crack", object_part = "screen"
    """
    # Placeholder: use regex/NLP model
    return issue_type, object_part

def analyze_images(image_paths, claim_object):
    """
    CV model or heuristic rules to detect damage.
    Returns list of dicts with image_id, issue_type, object_part, flags, severity.
    """
    results = []
    for path in image_paths.split(";"):
        image_id = os.path.basename(path).split(".")[0]
        # Placeholder: run detection model
        results.append({
            "image_id": image_id,
            "issue_type": "dent",   # example
            "object_part": "door", # example
            "flags": "none",
            "severity": "medium"
        })
    return results

def check_evidence_requirements(claim_object, issue_type, images):
    req = evidence_req[(evidence_req.claim_object == claim_object) &
                       (evidence_req.applies_to == issue_type)]
    if not req.empty and len(images) >= req.minimum_image_evidence.values[0]:
        return True, "Minimum evidence met"
    else:
        return False, "Insufficient images"

def apply_decision_logic(claim_details, image_results):
    issue_type, object_part = claim_details
    # Simplified logic
    if any(res["issue_type"] == issue_type and res["object_part"] == object_part for res in image_results):
        return "supported", "Damage visible in submitted images"
    elif all(res["issue_type"] == "none" for res in image_results):
        return "contradicted", "No damage visible in images"
    else:
        return "not_enough_information", "Images unclear or mismatch"

def add_user_history_flags(user_id):
    history = user_history[user_history.user_id == user_id]
    flags = []
    if not history.empty:
        if history.last_90_days_claim_count.values[0] > 3:
            flags.append("user_history_risk")
        if history.rejected_claim.values[0] > 0:
            flags.append("manual_review_required")
    return ";".join(flags) if flags else "none"

def estimate_severity(image_results):
    severities = [res["severity"] for res in image_results]
    if "high" in severities: return "high"
    if "medium" in severities: return "medium"
    if "low" in severities: return "low"
    if "none" in severities: return "none"
    return "unknown"

# Main pipeline
output_rows = []
for _, row in claims.iterrows():
    user_id = row["user_id"]
    claim_object = row["claim_object"]
    user_claim_text = row["user_claim"]
    image_paths = row["image_paths"]

    issue_type, object_part = extract_claim_details(user_claim_text)
    image_results = analyze_images(image_paths, claim_object)
    evidence_met, evidence_reason = check_evidence_requirements(claim_object, issue_type, image_results)
    claim_status, justification = apply_decision_logic((issue_type, object_part), image_results)
    risk_flags = add_user_history_flags(user_id)
    severity = estimate_severity(image_results)
    supporting_ids = ";".join([res["image_id"] for res in image_results if res["issue_type"] == issue_type]) or "none"
    valid_image = "true" if image_results else "false"

    output_rows.append([
        user_id, image_paths, user_claim_text, claim_object,
        evidence_met, evidence_reason, risk_flags,
        issue_type, object_part, claim_status, justification,
        supporting_ids, valid_image, severity
    ])

output_df = pd.DataFrame(output_rows, columns=[
    "user_id","image_paths","user_claim","claim_object",
    "evidence_standard_met","evidence_standard_met_reason",
    "risk_flags","issue_type","object_part","claim_status",
    "claim_status_justification","supporting_image_ids",
    "valid_image","severity"
])
output_df.to_csv("output.csv", index=False)
