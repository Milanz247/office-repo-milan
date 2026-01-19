SELECT 
    mu.user_name,
    mu.client_no,
    lg.nic,
    ci.client_no as client_info_exists,
    CASE 
        WHEN mu.client_no IS NOT NULL THEN 'ALREADY SYNCED (DUAL_USER)'
        ELSE 'NOT SYNCED YET'
    END as status
FROM cep_mobile_user mu
LEFT JOIN cep_lead_generator lg ON lg.mobile_user_id = mu.mobile_user_id  
LEFT JOIN pg_clientinfo ci ON ci.nic = lg.nic
WHERE mu.user_name = 'YOUR_USERNAME';
